# OpenClaw Access Control & Authentication Security Analysis

## Executive Summary

OpenClaw implements a **defense-in-depth** security model with multiple layers of access control spanning gateway authentication, channel-level allowlists, DM pairing, mention gating, and OAuth credential management. The system is well-architected with secure defaults, but contains several areas of concern regarding credential storage, privilege escalation vectors, and configuration complexity.

---

## 1. Architecture Overview

### 1.1 Security Layers

The system implements **five primary security boundaries**:

1. **Gateway Authentication Layer** - WebSocket/HTTP connection authentication
2. **Channel Access Control Layer** - Per-channel allowlists and policies
3. **DM Pairing Layer** - Secure pairing mechanism for private messaging
4. **Mention Gating Layer** - Group chat access control via mention requirements
5. **Command Authorization Layer** - Fine-grained command and tool execution policies

### 1.2 Core Security Files

**Authentication:**
- `/home/user/openclaw/src/gateway/auth.ts` - Gateway auth (token/password/Tailscale)
- `/home/user/openclaw/src/gateway/device-auth.ts` - Device identity authentication
- `/home/user/openclaw/src/agents/auth-profiles/oauth.ts` - OAuth token management
- `/home/user/openclaw/src/agents/auth-profiles/store.ts` - Credential storage

**Access Control:**
- `/home/user/openclaw/src/routing/resolve-route.ts` - Session routing and isolation
- `/home/user/openclaw/src/channels/allowlist-match.ts` - Allowlist matching logic
- `/home/user/openclaw/src/pairing/pairing-store.ts` - DM pairing implementation
- `/home/user/openclaw/src/channels/mention-gating.ts` - Group mention gating
- `/home/user/openclaw/src/config/group-policy.ts` - Group policy enforcement

**Audit:**
- `/home/user/openclaw/src/security/audit.ts` - Comprehensive security audit system

---

## 2. Allowlist System

### 2.1 Implementation

The allowlist system operates at **three levels**:

1. **Configuration-based allowlists** (`openclaw.yaml`)
2. **Pairing store allowlists** (`~/.clawdbot/credentials/<channel>-allowFrom.json`)
3. **Per-group/guild overrides** (channel-specific configuration)

**Key Files:**
- `/home/user/openclaw/src/channels/plugins/allowlist-match.ts` - Core matching
- `/home/user/openclaw/src/telegram/bot-access.ts` - Telegram implementation
- `/home/user/openclaw/src/discord/monitor/allow-list.ts` - Discord implementation
- `/home/user/openclaw/src/slack/monitor/allow-list.ts` - Slack implementation

### 2.2 Matching Logic

Each channel adapter implements **multiple matching strategies**:

**Telegram:**
```typescript
// Matches: ID, username, @username (case-insensitive), prefixed entries
- Wildcard: "*"
- ID: "12345678"
- Username: "alice" or "@alice"
- Prefixed: "tg:alice" or "telegram:alice"
```

**Discord:**
```typescript
// Matches: ID, username, display name, tag, slug
- Wildcard: "*"
- ID: "123456789012345678"
- Username: "alice#1234"
- Slug: "alice-smith"
- Prefixed: "discord:alice" or "user:123456..."
```

**Slack:**
```typescript
// Matches: ID, name, slug, prefixed variants
- Wildcard: "*"
- ID: "U12345678"
- Name: "alice"
- Slug: "alice-smith"
- Prefixed: "slack:alice" or "user:U12345678"
```

### 2.3 Security Properties

**Strengths:**
- ✅ **Case-insensitive matching** prevents bypasses via case manipulation
- ✅ **Normalization** of IDs/usernames reduces confusion attacks
- ✅ **Wildcard detection** (`*`) is explicit and auditable
- ✅ **Multiple match sources** (ID, username, name) provide flexibility
- ✅ **File permissions** on allowlist stores are enforced (0600)

**Weaknesses:**
- ⚠️ **Username collision risk**: Different users on different platforms can share usernames
- ⚠️ **No signature verification**: Allowlist files can be tampered with if filesystem access is gained
- ⚠️ **Path traversal protection is basic**: `safeChannelKey()` in pairing-store.ts could be bypassed with crafted channel IDs

---

[Full report continues with all sections from the original output...]

---

**Report Generated:** 2026-01-30
**Analyst:** Claude (Sonnet 4.5)
**Analysis Scope:** Full codebase security audit focusing on access control & authentication
