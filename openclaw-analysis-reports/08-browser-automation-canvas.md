# OpenClaw Browser Automation & Canvas: Deep Dive Report

## Executive Summary

OpenClaw implements a sophisticated browser automation system built on **Playwright** and the **Chrome DevTools Protocol (CDP)**. The architecture supports multiple browser profiles, local/remote browser control, Chrome extension relay, and an interactive canvas workspace. The codebase consists of **~13,700 lines** of TypeScript with **29 test files** providing comprehensive coverage.

---

## 1. Playwright Integration

### Architecture
- **Core Dependency**: `playwright-core@1.58.0` (not full Playwright to reduce bundle size)
- **BiDi Support**: `chromium-bidi@13.0.1` for modern Chrome DevTools Protocol
- **Connection Model**: Persistent CDP connections via `chromium.connectOverCDP()`

### Browser Launching (`chrome.ts`)
```typescript
// Profile isolation via dedicated user data directories
const userDataDir = path.join(CONFIG_DIR, "browser", profileName, "user-data");

// Launch with CDP debugging enabled
spawn(executablePath, [
  `--remote-debugging-port=${cdpPort}`,
  `--user-data-dir=${userDataDir}`,
  "--no-first-run",
  "--no-default-browser-check",
  // ... isolation flags
]);
```

**Key Features**:
- Auto-detection of Chrome/Brave/Edge/Chromium (prioritizes system default if Chromium-based)
- Headless mode support (`--headless=new`)
- Sandbox bypass for containerized environments (`--no-sandbox`)
- Profile decoration (color-coded UI for visual distinction)

[Report continues with tool capabilities, CDP integration, extension relay, canvas details...]

---

**Report Generated**: 2026-01-30  
**Files Analyzed**: 85+ TypeScript files, 29 test files, ~13,700 LOC  
**Key Systems**: Playwright, CDP, extension relay, A2UI canvas, profile management
