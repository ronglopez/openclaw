# OpenClaw Mobile Architecture: Comprehensive Deep Dive

## Executive Summary

OpenClaw's mobile apps (iOS and Android) transform smartphones into **powerful AI agent nodes** that extend the gateway's capabilities with device-specific actions. Both apps follow a **dual-session architecture**: one WebSocket connection for the operator UI (chat/control) and another for node tool invocations (camera, screen recording, canvas). The mobile apps are fully native (SwiftUI on iOS, Jetpack Compose on Android) with embedded WebView-based canvas rendering, voice wake capabilities, and sophisticated TTS integration.

---

## 1. iOS Architecture

### SwiftUI App Structure
- **Entry Point:** `OpenClawApp.swift` - SwiftUI `@main` app with two root `@State` objects:
  - `NodeAppModel` - main app state container
  - `GatewayConnectionController` - manages gateway connection lifecycle
- **Modern Swift Concurrency:** Uses Swift 6 with strict concurrency checking
- **Observation Framework:** **Exclusively uses `@Observable` and `@Bindable`** from the Observation framework (no `ObservableObject`)
  - `NodeAppModel` is `@MainActor @Observable`
  - `VoiceWakeManager`, `TalkModeManager` use `@Observable`
  - Clean, modern state management without Combine

### Key Components
```
OpenClawApp
├── NodeAppModel (@Observable)
│   ├── CameraController (actor)
│   ├── ScreenController (@Observable)
│   ├── VoiceWakeManager (@Observable)
│   ├── TalkModeManager (@Observable)
│   └── GatewayNodeSession (actor)
├── GatewayConnectionController
└── RootCanvas (SwiftUI View)
```

---

## 2. Android Architecture

### Kotlin App Structure
- **Entry Point:** `MainActivity` (ComponentActivity) with Jetpack Compose
- **ViewModel Pattern:** `MainViewModel` holds all app state
- **Modern Android:** Minimum SDK 31 (Android 12), target SDK 36
- **Jetpack Compose UI:** Material 3 design system

### Key Components
```
MainActivity
├── MainViewModel (ViewModel)
│   ├── NodeRuntime
│   │   ├── GatewaySession (operator + node)
│   │   ├── CameraCaptureManager
│   │   ├── ScreenRecordManager
│   │   ├── VoiceWakeManager
│   │   └── TalkModeManager
│   └── Preferences (SecurePrefs)
└── NodeForegroundService
```

[Report continues with node mode, device actions, gateway connection, canvas, voice wake, talk mode...]

---

**Report Generated**: 2026-01-30  
**Codebase Version**: iOS `2026.1.27-beta.1`, Android `2026.1.29`  
**Files Analyzed**: 50+ source files across iOS, Android, shared components
