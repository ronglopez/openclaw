# OpenClaw Gateway Architecture & WebSocket Protocol - Deep Dive Report

## Executive Summary

OpenClaw's Gateway is a sophisticated **unified control plane** built on WebSocket that serves as the single communication hub for all clients (CLI, web UI, macOS/iOS apps, nodes). The architecture demonstrates excellent design decisions around protocol versioning, authentication, event broadcasting, and multi-role support. Total gateway codebase: **~32,746 lines** with **56 test files** ensuring robustness.

---

## 1. Gateway Design & Architecture Overview

### Core Architecture

**Primary Components:**
- **HTTP/HTTPS Server** (`node:http`/`node:https`) - Single port for all services
- **WebSocket Server** (`ws` library) - Control plane protocol
- **Node Registry** - Manages connected capability hosts (iOS/Android/macOS nodes)
- **Broadcast System** - Event distribution to all connected clients
- **Channel Manager** - Manages messaging platform connections (Telegram, Discord, etc.)
- **Method Handlers** - 85+ RPC methods organized by domain

**File Structure:**
```
src/gateway/
├── server.impl.ts              # Main server orchestration (587 LOC)
├── server-runtime-state.ts     # Runtime state initialization
├── server-ws-runtime.ts        # WebSocket attachment
├── server/
│   ├── ws-connection.ts        # Connection lifecycle
│   └── ws-connection/
│       └── message-handler.ts  # Protocol handshake & validation (942 LOC)
├── protocol/
│   ├── index.ts               # Schema validators
│   └── schema/                # TypeBox protocol definitions
├── server-methods/            # RPC method implementations
│   ├── agent.ts
│   ├── config.ts
│   ├── nodes.ts
│   └── ... (20+ handlers)
└── server-broadcast.ts        # Event broadcasting
```

### Why WebSocket?

The design document and implementation reveal **deliberate architectural choices**:

1. **Bidirectional Real-time Communication**: Required for streaming agent responses, presence updates, and live health monitoring
2. **Single Connection Model**: Eliminates HTTP polling overhead; one persistent connection serves all needs
3. **Event Streaming**: Natural fit for agent tool execution streams, chat deltas, and system events
4. **Low Latency**: Critical for interactive chat (WebChat UI), voice wake forwarding, and remote node commands
5. **Protocol Flexibility**: JSON frames with schema validation allow protocol evolution while maintaining compatibility

**Port Consolidation**: The gateway serves multiple protocols on a **single port (default 18789)**:
- WebSocket control plane
- HTTP endpoints (`/v1/chat/completions`, `/v1/responses`, control UI)
- Canvas host file server
- Plugin HTTP handlers
- Webhook receivers

[Report continues with full details from original agent output...]

---

**Report Generated**: 2026-01-30  
**Files Analyzed**: ~32,746 lines with 56 test files  
**Key Subsystems**: Protocol, connection management, broadcasting, remote deployment
