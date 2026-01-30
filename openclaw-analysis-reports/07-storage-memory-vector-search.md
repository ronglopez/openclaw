# OpenClaw Storage, Memory & Vector Search Deep Dive

## Executive Summary

OpenClaw employs a **local-first, file-based architecture** with a well-organized directory structure centered around `~/.openclaw/` (with legacy support for `~/.clawdbot/`). The system uses **sqlite-vec** for efficient vector similarity search combined with FTS5 for hybrid semantic + keyword search.

---

## 1. Storage Architecture Overview

### Directory Structure
```
~/.openclaw/
├── openclaw.json                    # Main configuration file
├── credentials/                      # Authentication credentials
│   ├── oauth.json                   # Web provider OAuth tokens
│   └── whatsapp/default/creds.json  # WhatsApp Web session
├── agents/{agentId}/
│   └── sessions/
│       ├── sessions.json            # Session metadata store
│       └── *.jsonl                  # Session transcripts (append-only)
└── workspace/                       # Default workspace directory
```

### Key Design Principles
1. **Local-first**: All data persists on local filesystem
2. **File permissions**: Sensitive files (credentials, session stores) use mode `0o600`
3. **Atomic writes**: Uses temp files + rename for crash safety (except Windows)
4. **Legacy migration**: Automatic migration from `.clawdbot` → `.openclaw`

---

## 2. Session Persistence

Sessions use a **dual-storage model**: metadata in JSON + transcripts in JSONL.

### Session Transcripts (JSONL Format)
- **Location**: `~/.openclaw/agents/{agentId}/sessions/{sessionId}.jsonl`
- **Format**: Newline-delimited JSON (JSONL) using `@mariozechner/pi-coding-agent`
- **Append-only**: Messages appended to preserve complete conversation history
- **Header**: First line contains session metadata (version, ID, timestamp, cwd)

[Report continues with config storage, credentials, vector DB, semantic search, embeddings...]

---

**Report Generated**: 2026-01-30  
**Key Systems**: JSONL sessions, JSON5 config, sqlite-vec, hybrid search, embedding cache
