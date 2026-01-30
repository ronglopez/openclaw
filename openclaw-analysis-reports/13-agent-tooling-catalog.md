# OpenClaw Complete Agent Tooling System: Deep-Dive Technical Analysis

## Executive Summary

OpenClaw implements a **massive, production-grade agent tooling system** with 25+ distinct tools spanning runtime execution, file manipulation, web access, browser automation, device control, messaging, session management, and automation infrastructure.

**Statistics:**
- **25+ core tools**, expandable via plugins
- **100+ distinct actions** across all tools
- **20+ implementation files**, 8,000+ LOC
- **8-layer policy hierarchy** for fine-grained control
- **Multi-host execution**: sandbox/gateway/node routing
- **Production-grade**: Approval workflows, SSRF protection, sandbox isolation

## Complete Tool Catalog

### 1. Runtime Execution Tools

#### **exec** (Bash Tool)
**Location:** `src/agents/bash-tools.exec.ts` (~2,500 LOC)

The crown jewel for shell command execution with sophisticated multi-host execution:

**Capabilities:**
- **Multi-host**: `sandbox` (Docker), `gateway` (local), `node` (remote device)
- **Security modes**: `deny`, `allowlist`, `full`
- **Approval system**: Interactive with auto-allowlist, 120s timeout
- **PTY support**: Full pseudo-terminal with DSR handling
- **Background execution**: Auto-background after 10s (configurable)
- **Node routing**: Remote execution via gateway protocol
- **Safe bins**: Allowlist bypassing approval (git, npm, ls, etc.)

**Approval Workflow:**
1. Command submitted → security evaluation
2. Requires approval? → Generate 8-char slug
3. Send system event to requester
4. Wait up to 120s for response
5. On approval: execute + add to allowlist
6. On timeout/reject: abort

#### **process** (Process Management)
**Location:** `src/agents/bash-tools.process.ts`

**Actions (10):** list, poll, log, write, send-keys, submit, paste, kill, clear, remove

**Key Features:**
- Session-scoped visibility
- Job TTL for cleanup
- Full stdin control
- DSR handling for cursor positioning

### 2. File System Tools

#### **read**, **write**, **edit**
**Source:** `@mariozechner/pi-coding-agent`

**Features:**
- Absolute path resolution
- Line offset/limit (2000 default)
- Sandbox boundary enforcement
- UTF-8 encoding

#### **apply_patch**
**Location:** `src/agents/apply-patch.ts`

Multi-file patch application for OpenAI models:

**Operations:**
- `*** Add File`: Create with contents
- `*** Delete File`: Remove
- `*** Update File`: Apply hunks
- `*** Move to`: Rename/move

**Restrictions:** OpenAI models only, allowlist-based gating

### 3. Web Tools

#### **web_search**
**Location:** `src/agents/tools/web-search.ts`

**Providers:**
- **Brave Search API**: Region, language, freshness filters
- **Perplexity Sonar**: AI-synthesized with citations

**Features:**
- 15-minute self-cleaning cache
- SSRF protection
- Result count: 1-10 (default 5)
- Date range filtering

#### **web_fetch**
**Location:** `src/agents/tools/web-fetch.ts`

HTML → Markdown/Text extraction:

**Modes:**
- `markdown`: Readability + turndown
- `text`: Pure text extraction

**Features:**
- Firecrawl API integration (opt-in)
- Mozilla Readability
- Redirect following (max 3 hops)
- SSRF protection with hostname pinning
- 50k char truncation
- 15-minute cache

### 4. Browser Tool

#### **browser**
**Location:** `src/agents/tools/browser-tool.ts` (715 LOC)

Playwright-backed browser automation with 20+ actions:

**Architecture:**
- **Multi-target**: sandbox/host/node routing
- **100+ profiles**: Ports 18800-18899
- **Chrome relay**: Takeover user's existing tabs

**Actions (16):**
1. status, start, stop, profiles
2. tabs, open, focus, close
3. snapshot (AI/aria formats)
4. screenshot (page/element/full)
5. navigate, console, pdf
6. upload (file chooser hook)
7. dialog (alert/confirm handling)
8. act (automation actions)

**Act Automation (11 kinds):**
- click, type, press, hover, drag
- select, fill, resize, wait
- evaluate, close

**Chrome Extension Relay:**
- User clicks toolbar on tab → attached
- All browser actions route to that tab
- Requires `profile="chrome"`

**Node Proxy:**
When `target=node`:
1. Resolve node ID
2. Call `node.invoke` with `browser.proxy`
3. Gateway routes to node
4. Results + files returned as base64

### 5. Canvas Tool

#### **canvas**
**Location:** `src/agents/tools/canvas-tool.ts`

Node canvas presentation + A2UI rendering:

**Actions (7):**
1. present: Show with URL + placement
2. hide: Dismiss
3. navigate: Load URL
4. eval: Execute JavaScript
5. snapshot: Capture PNG/JPEG
6. a2ui_push: Push JSONL updates
7. a2ui_reset: Clear state

**A2UI System:**
- Agent-to-UI reactive framework
- JSONL-based state updates
- Real-time dashboards, live data viz

### 6. Nodes Tool

#### **nodes**
**Location:** `src/agents/tools/nodes-tool.ts`

Device discovery + paired node control:

**Actions (12):**
1. status: List connected nodes
2. describe: Detailed node info
3. pending: Pairing requests
4. approve/reject: Pairing management
5. notify: System notifications
6. camera_snap: Photo (front/back/both)
7. camera_list: Enumerate cameras
8. camera_clip: Video recording
9. screen_record: Screen capture
10. location_get: GPS query
11. run: Execute command on node

**Camera Features:**
```typescript
{
  facing: "front" | "back" | "both",
  maxWidth?: number,
  quality?: number,
  delayMs?: number,
  deviceId?: string,
  format: "jpg" | "png"
}
```

**Screen Recording:**
```typescript
{
  durationMs: 10000,
  fps: 10,
  screenIndex: 0,
  includeAudio: true,
  format: "mp4"
}
```

**Location Accuracy:** coarse (~100m), balanced (~50m), precise (<10m)

### 7. Image Tool

#### **image**
**Location:** `src/agents/tools/image-tool.ts`

Vision model inference:

**Model Selection:**
1. Explicit config
2. Provider pairing (match primary)
3. Cross-provider fallback

**Supported Providers:**
- OpenAI: `gpt-5-mini`
- Anthropic: `claude-opus-4-5`
- MiniMax: `MiniMax-VL-01`

**Input Modes:** File path, file:// URL, data: URL, HTTP/HTTPS URL

### 8. Message Tool

#### **message**
**Location:** `src/agents/tools/message-tool.ts`

Unified cross-channel messaging with **50+ actions**:

**Core Actions:**
- send, broadcast, poll, react, reactions, read
- edit, unsend, reply, sendWithEffect
- delete, pin, unpin, list-pins
- permissions, thread-create, thread-list, thread-reply
- search, sticker, member-info, role-info
- emoji-list, emoji-upload, sticker-upload
- role-add, role-remove, channel-info, channel-list
- voice-status, event-list, event-create
- timeout, kick, ban

**Channels:**
- Core: Telegram, Discord, Slack, Signal, iMessage, WhatsApp
- Extensions: MS Teams, Matrix, Zalo, Voice Call

**Advanced Features:**
- Telegram inline keyboards
- Adaptive Cards (Teams/Outlook)
- iMessage effects (invisible-ink, balloons)
- Slack auto-threading
- Cross-context routing
- Silent delivery
- Quote text (Telegram)

**Action Filtering:**
Tool description adapts based on current channel capabilities.

### 9. Session Management Tools

#### **sessions_list**
**Location:** `src/agents/tools/sessions-list-tool.ts`

Enumerate agent sessions:

**Filters:**
- `kinds`: main/group/cron/hook/node/other
- `limit`: Max results
- `activeMinutes`: Recency filter
- `messageLimit`: Last N messages (0-20)

**Sandbox Visibility:** Only spawned sessions when sandboxed

#### **sessions_history**
Fetch chat history with tool filtering

#### **sessions_send**
**Location:** `src/agents/tools/sessions-send-tool.ts`

Cross-session messaging (agent-to-agent):

**Ping-Pong System:**
When `timeoutSeconds > 0`:
1. Send → target session
2. Wait for completion
3. Extract reply
4. Background announce-back flow
5. Continue up to `maxPingPongTurns` (default 3)

**Execution Modes:**
- `timeoutSeconds > 0`: Wait for response
- `timeoutSeconds = 0`: Fire-and-forget

#### **sessions_spawn**
**Location:** `src/agents/tools/sessions-spawn-tool.ts`

Background sub-agent execution:

**Features:**
- Isolated session with UUID
- Model override
- Thinking level override
- Timeout control
- Announce-back lifecycle
- Cleanup policy

**Lifecycle:**
1. Create `agent:{agentId}:subagent:{uuid}`
2. Patch model
3. Build subagent system prompt
4. Submit with `lane=subagent`
5. Register in subagent registry
6. Return `{status: "accepted", runId, childSessionKey}`

**Announce-Back:**
On completion, deliver to requester with `runId`, `childSessionKey`, reply text.

#### **session_status**, **agents_list**
Query metadata and enumerate agents.

### 10. Memory Tools

#### **memory_search**
**Location:** `src/agents/tools/memory-tool.ts`

Semantic search over memory files:

**Search Scope:**
- `MEMORY.md`
- `memory/*.md`
- Configured extra paths
- Optional: Session transcripts

**Features:**
- Vector embeddings (provider configurable)
- Semantic similarity ranking
- Min score threshold
- Snippet extraction with line numbers

#### **memory_get**
Safe snippet read from memory files with path validation.

### 11. Automation Tools

#### **cron**
**Location:** `src/agents/tools/cron-tool.ts`

Scheduled job management:

**Actions (8):** status, list, add, update, remove, run, runs, wake

**Context Injection:**
`contextMessages` parameter fetches recent chat history and appends to job text.

#### **gateway**
**Location:** `src/agents/tools/gateway-tool.ts`

Gateway self-management:

**Actions (6):**
1. restart: SIGUSR1 with delay
2. config.get: Fetch config
3. config.schema: JSON Schema
4. config.apply: Replace config
5. config.patch: Merge partial
6. update.run: npm update + restart

**Config Patching:**
- Hash-based conflict detection
- Deep merge semantics
- Auto-restart
- Notification to originating session

## Architecture & Implementation

### Tool Definition Framework

```typescript
interface AgentTool<TParams, TDetails> {
  name: string
  label: string
  description: string
  parameters: TypeBoxSchema<TParams>
  execute: (
    toolCallId: string,
    params: TParams,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback
  ) => Promise<AgentToolResult<TDetails>>
}
```

**Result Format:**
```typescript
type AgentToolResult = {
  content: Array<
    | { type: "text", text: string }
    | { type: "image", data: string, mimeType: string }
  >
  details?: TDetails
}
```

### Pi Agent Core Integration

**Adapter:** `src/agents/pi-tool-definition-adapter.ts`

Converts OpenClaw tools → Pi Coding Agent `ToolDefinition[]`:
- Signature adaptation (parameter order)
- Error normalization + logging
- AbortError preservation

### Tool Policy System

**Location:** `src/agents/tool-policy.ts`

**8-Layer Hierarchy:**
1. Profile policy
2. Provider profile policy
3. Global allow/deny
4. Provider allow/deny
5. Agent allow/deny
6. Agent provider policy
7. Group policy
8. Sandbox policy

**Policy DSL:**
```typescript
{
  allow?: string[],  // Allowlist
  deny?: string[]    // Denylist
}
```

**Tool Groups:**
```typescript
{
  "group:memory": ["memory_search", "memory_get"],
  "group:web": ["web_search", "web_fetch"],
  "group:fs": ["read", "write", "edit", "apply_patch"],
  "group:runtime": ["exec", "process"],
  "group:sessions": ["sessions_list", "sessions_history", ...],
  "group:ui": ["browser", "canvas"],
  "group:automation": ["cron", "gateway"],
  "group:messaging": ["message"],
  "group:nodes": ["nodes"]
}
```

**Profiles:**
- `minimal`: `session_status` only
- `coding`: fs + runtime + sessions + memory + image
- `messaging`: messaging + sessions (list/history/send/status)
- `full`: No restrictions

### Tool Schema Compatibility

**Challenge:** Multiple LLM providers with different JSON Schema quirks

**Solutions:**
1. Avoid top-level unions (OpenAI rejects)
2. Flattened schemas with discriminator
3. No TypeBox Type.Union at root
4. Claude compatibility patches
5. Gemini cleanup

### Security & Gating

#### Exec Approval System

**Storage:** `~/.openclaw/exec-approvals.json`

**Evaluation:**
```typescript
function requiresExecApproval(params: {
  command, host, security, ask, safeBins, approvals
}): boolean
```

**Safe Bins:** git, npm, ls, pwd, cat, echo, grep, find, etc.

#### Elevated Mode

Execute commands with elevated permissions on gateway:

**Config:**
```typescript
{
  enabled: true,
  allowed: true,
  defaultLevel: "ask"
}
```

**Security:** Bypasses sandbox, routes to gateway, requires allowance

#### Sandbox Isolation

**Docker Sandbox:**
```typescript
{
  containerName,
  workspaceDir,
  containerWorkdir,
  env
}
```

**Path Validation:** No traversal outside sandbox root, canonical resolution, symlink handling

## Advanced Features

### Multi-Profile Browser Support

- **100 profiles** supported (ports 18800-18899)
- Profile ID → Port mapping via hash
- Persistence in `~/.openclaw/browser/profiles/`

**Chrome Extension Relay:**
- Extension creates WebSocket on port 18801
- User attaches tab via toolbar icon
- Actions route to attached tab
- Preserves logged-in sessions

### Background Process Management

**Process Registry:**
```typescript
{
  id: string,
  command, pid, startedAt,
  backgrounded: boolean,
  aggregated: string,
  exited, exitCode, exitSignal,
  scopeKey: string  // Agent isolation
}
```

**Lifecycle:** exec → output stream → auto-background (10s) → poll → exit → TTL cleanup

### Cross-Session Communication

**Ping-Pong Protocol:**
```
[Agent A] sessions_send(B, "Hello", timeout=30)
  ↓
[Gateway] Route to Agent B
  ↓
[Agent B] Process → reply
  ↓
[Agent A] Receives reply
  ↓
[Background] Announce-back flow (up to maxPingPongTurns=3)
```

**Policy Gating:**
```json
{
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": ["*"]
    }
  }
}
```

### Sub-Agent Spawning

**Session Hierarchy:**
```
agent:alice
  ├─ agent:alice:subagent:uuid-1
  ├─ agent:alice:subagent:uuid-2
  └─ agent:alice:thread:123
```

**Spawning Flow:**
1. sessions_spawn({task, label, model})
2. Create child session
3. Build subagent system prompt
4. Submit with lane=subagent
5. Register in subagent registry
6. Return {status: "accepted"}

### Gateway Self-Management

**Update Flow:**
1. `gateway({action: "update.run"})`
2. Execute `npm update -g openclaw@latest`
3. Write restart sentinel
4. Schedule SIGUSR1
5. Graceful shutdown
6. exec(process.argv) → restart
7. Deliver notification

**Config Patching:**
Agent modifies config without manual editing:
- Hash-based conflict detection
- Deep merge
- Schema validation
- Atomic write + restart

## Comparison to Other Frameworks

### vs. LangChain
**OpenClaw +:** Multi-host, approval workflows, cross-session, node control  
**LangChain +:** 10x more community tools, multi-language

### vs. AutoGPT
**OpenClaw +:** Gateway, sandbox, multi-channel, sessions, browser, nodes  
**AutoGPT +:** Autonomous loops, long-term memory

### vs. Semantic Kernel
**OpenClaw +:** Unified messaging, cross-session, multi-host, browser  
**Semantic Kernel +:** Azure integration, multi-language, prompt templating

### vs. Claude Code
**OpenClaw +:** 4x more categories, messaging, browser, infrastructure, multi-agent  
**Claude Code +:** Simplicity (6 tools vs 25+), coding-focused, official

## Strengths & Weaknesses

### Strengths
1. **Comprehensive**: 25+ tools, 100+ actions
2. **Production-grade**: Gateway, sandbox, approval, policy
3. **Multi-channel**: 50+ actions across 8+ platforms
4. **Cross-session**: Agent-to-agent, subagent spawning
5. **Browser**: Full Playwright (20+ actions)
6. **Node infrastructure**: Device pairing, camera, location
7. **Policy flexibility**: 8-layer hierarchy
8. **Security**: Sandbox, SSRF, approval, elevated gating
9. **Self-management**: Update/restart/config
10. **Automation**: Cron, wake events, background processes

### Weaknesses
1. **Complexity**: Steep learning curve
2. **TypeScript-only**: No Python/C#
3. **Documentation debt**: Minimal tool docs
4. **Tool overlap**: Some duplication
5. **Schema quirks**: Flattened to work around providers
6. **Approval UX**: System events (not ideal)
7. **Error handling**: Inconsistent formats
8. **Testing**: Some tools lack comprehensive tests

### Technical Debt
1. Browser tool: 715 LOC (should split)
2. Message tool schema: 50+ optional params
3. Exec tool: 2,500 LOC (should modularize)
4. Tool policy: 8-layer resolution (fragile)

### Missing Capabilities
1. Multi-modal input (audio/video)
2. Database tools (SQL, vector, graph)
3. Cloud integration (AWS/GCP/Azure)
4. Monitoring/observability
5. Human-in-the-loop (general-purpose)
6. Parallel tool execution
7. Tool versioning

## Recommendations

### High-Priority
1. Split large tools (browser, exec, message)
2. Unify error format
3. Improve documentation
4. Simplify policy (8 → 4 layers)
5. Add tool tests (80%+ coverage)
6. Schema validation (runtime)
7. Approval UX (interactive UI)

### Future Enhancements
1. Database tools
2. Cloud integrations
3. Multi-modal input
4. Tool versioning
5. Parallel execution
6. Per-tool sandboxing
7. Observability hooks

## Conclusion

OpenClaw's Agent Tooling System is a **production-grade, comprehensive framework** rivaling commercial platforms in scope while maintaining flexibility through sophisticated policy system. With 25+ core tools spanning all major categories, it provides capabilities far beyond basic coding assistants.

The architecture demonstrates deep sophistication: multi-host routing, Docker isolation, approval workflows, cross-session communication, sub-agent spawning, gateway self-management. The 8-layer policy hierarchy enables fine-grained control.

However, complexity is the trade-off. Learning curve is steep, documentation minimal, some tools too large. Future work should focus on modularization, testing, unified errors, simplified policies.

Despite challenges, OpenClaw stands as one of the most complete open-source agent frameworks, offering capabilities (multi-channel messaging, node control, cross-agent communication) that commercial platforms lack.

**Final Assessment: 8/10**
- Breadth: 9/10
- Depth: 8/10
- Architecture: 9/10
- Security: 8/10
- Usability: 6/10
- Documentation: 4/10

**Files Analyzed:** 20+ tool files, 8,000+ LOC, complete subsystem
