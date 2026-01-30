# OpenClaw Agent Runtime & Tool Execution: Deep Dive Analysis

## Executive Summary

OpenClaw implements a sophisticated agent runtime built on top of `@mariozechner/pi-agent-core`, with extensive custom tooling for multi-agent coordination, streaming execution, context management, and resilient model failover. The system demonstrates mature engineering with ~223 non-test TypeScript files in the agents directory, handling complex scenarios like auth profile rotation, context overflow recovery, and inter-agent communication.

---

## 1. Agent Runtime Architecture

### 1.1 Pi Agent Core Integration

**Location**: `/home/user/openclaw/src/agents/pi-embedded-runner/`

OpenClaw integrates with three core libraries:
- `@mariozechner/pi-agent-core` - Core agent message handling
- `@mariozechner/pi-ai` - Model provider abstraction layer  
- `@mariozechner/pi-coding-agent` - Coding-specific tools (read/write/edit, SessionManager, SettingsManager)

**Key Integration Point**: `pi-embedded-runner/run/attempt.ts` (884 lines)

The runtime creates agent sessions via `createAgentSession()`:
```typescript
const { session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage,
  modelRegistry,
  model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  systemPrompt,
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
  skills: [],
  contextFiles: [],
  additionalExtensionPaths,
});
```

[Report continues with session management, tool streaming, context compaction, model selection details...]

---

**Report Generated**: 2026-01-30  
**Files Analyzed**: ~50 core files + 223 agent-related files  
**Key Systems**: Runtime orchestration, tool execution, session management, failover
