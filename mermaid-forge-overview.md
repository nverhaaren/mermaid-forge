# Mermaid Forge — Project Overview

## Problem Statement

The official Mermaid Chart editor (mermaid.ai) and the open-source Mermaid Live Editor (mermaid.live) are both desktop-first web applications. On mobile devices — particularly Android phones — the UI breaks down: buttons overlap, text fields become unusably narrow, and the split-pane editor/preview layout doesn't adapt to small screens. This makes it impractical to create or edit mermaid diagrams on the go, despite the text-based syntax being well-suited to mobile input.

Additionally, the AI-assisted diagram generation features in Mermaid Chart are tied to their own backend and pricing. There is no existing tool that combines a mobile-friendly mermaid editing experience with bring-your-own-key (BYOK) AI integration, letting users choose their preferred LLM provider and pay only for what they use.

## Target Use Case

A developer or technical user who wants to:

- Create and edit mermaid diagrams from a phone or tablet with a responsive, touch-friendly interface
- Use AI (via their own API key) to generate or modify diagrams from natural language descriptions
- Maintain a personal library of diagrams with persistent local storage
- Sync diagrams to a GitHub repository for version control, cross-device access, and portability
- Export diagrams as SVG (for embedding in docs/presentations) or raw `.mmd` (for use in other mermaid-compatible tools)

The primary persona is a solo developer or technical professional. Collaboration features, team management, and real-time co-editing are explicitly out of scope.

## Existing Landscape

| Tool | Mobile UX | BYOK AI | Local Storage | GitHub Sync |
|------|-----------|---------|---------------|-------------|
| Mermaid Chart (mermaid.ai) | Poor — layout breaks on narrow screens | No — uses built-in AI backend | Cloud (account-based) | No |
| Mermaid Live Editor (mermaid.live) | Usable but cramped, no AI features | No | None (URL-encoded sharing) | No |
| AI Mermaid (Android app) | Exists but poorly reviewed; WebView wrapper with ads | No | Unknown | No |
| MermAIdrawing (iPad only) | iPad-optimized | OpenAI only | iCloud | No |
| Mermaid Mind (web) | Not mobile-optimized | No (uses Gemini backend) | No | No |

No existing tool combines mobile-first UX, multi-provider BYOK AI, local persistence, and GitHub sync.

## Approach: Progressive Web App (PWA)

After evaluating native Android, cross-platform frameworks, and web-based approaches, a Progressive Web App was selected as the best fit.

### Why PWA over native

- **Single codebase** — works on Android, iOS, tablets, and desktop from one set of HTML/JS/CSS files
- **No app store overhead** — no developer accounts, review processes, or update delays
- **Mermaid.js is web-native** — the rendering library is JavaScript; any native approach would still need a WebView for rendering, so going fully web avoids an unnecessary native shell
- **Offline capable** — service workers cache all static assets; editing and rendering work without a network connection
- **Installable** — Chrome on Android supports "Add to Home Screen" with standalone display mode, giving a native-app-like experience
- **Zero backend** — all API calls (AI providers, GitHub) go directly from the browser to the provider; no server to maintain, no infrastructure costs

### Why not native Android (Kotlin/Compose)

Mermaid.js has no native Kotlin port, so rendering would require a WebView regardless. A native wrapper adds build complexity (Gradle, signing, Play Store) without meaningful UX benefits for this use case. If Play Store distribution is later desired, a Trusted Web Activity (TWA) wraps the PWA in an APK with minimal effort.

### Why not Tauri Mobile / Rust-based

Tauri Mobile is still maturing and adds significant complexity. The core logic (text editing, API calls, IndexedDB storage) is naturally web-tier work. Rust would be beneficial if there were compute-heavy client-side processing, but mermaid rendering is handled by the mermaid.js library and there's no need for a native performance layer.

## Architecture

```
┌──────────────────────────────────────────────┐
│                  Browser                      │
│                                               │
│  ┌─────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ Editor  │  │ Mermaid  │  │  AI Prompt  │ │
│  │  (text) │→ │ .js      │→ │  (BYOK)     │ │
│  │         │  │ renderer │  │             │ │
│  └────┬────┘  └──────────┘  └──────┬──────┘ │
│       │                            │         │
│  ┌────▼────────────────────────────▼──────┐  │
│  │           IndexedDB                     │  │
│  │  diagrams: {id, name, code, ghPath,    │  │
│  │            ghSha, createdAt, updatedAt} │  │
│  └────────────────┬───────────────────────┘  │
│                   │                           │
│  ┌────────────────▼───────────────────────┐  │
│  │         GitHub Contents API             │  │
│  │  PUT/GET /repos/{owner}/{repo}/contents │  │
│  │  Auth: fine-grained PAT (Contents R/W) │  │
│  └────────────────────────────────────────┘  │
│                                               │
│  ┌──────────────────────────────────────────┐ │
│  │         Service Worker                    │ │
│  │  Caches: HTML, JS, CSS, fonts, mermaid   │ │
│  │  Strategy: cache-first for static assets │ │
│  └──────────────────────────────────────────┘ │
│                                               │
│  localStorage: settings, API keys, theme      │
└──────────────────────────────────────────────┘
```

All state is client-side. Settings and API keys live in localStorage. Diagrams live in IndexedDB. GitHub sync is manual push/pull against the Contents API using a fine-grained PAT scoped to a single repository with only Contents: Read and Write permission.

## Roadmap

### Phase 1: Functional local tool
Make the existing prototype a fully usable local diagram editor.

- IndexedDB-backed multi-diagram storage with create/switch/delete
- Editable diagram names
- Import `.mmd` files, export as SVG or `.mmd`
- BYOK AI generation (Anthropic, OpenAI, OpenRouter)
- Auto-save, auto-render, configurable mermaid themes
- Template library for common diagram types

### Phase 2: PWA + GitHub sync
Make it installable, offline-capable, and syncable.

- Web app manifest for Add to Home Screen / standalone mode
- Service worker for offline caching
- GitHub sync: push current diagram, pull all diagrams from repo
- Guided fine-grained PAT creation flow in settings
- Sync status indicators per diagram (local / synced)

### Phase 3: UX polish
Improve the editing experience for daily use.

- CodeMirror 6 integration with mermaid syntax highlighting
- Pinch-to-zoom on diagram preview (panzoom library or touch gesture handling)
- Undo/redo
- Diagram search/filter in the list
- Light/dark theme toggle for the preview pane
- Share via Web Share API

### Phase 4: Distribution (optional)
Play Store presence if desired.

- Trusted Web Activity (TWA) wrapper via Bubblewrap
- App icons and splash screens
- Play Store listing

### Non-goals

- Real-time collaboration or multi-user editing
- Server-side rendering or backend infrastructure
- Team/org management features
- Support for diagram formats other than mermaid.js
