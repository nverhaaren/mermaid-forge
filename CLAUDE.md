# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mermaid Forge is a mobile-first PWA for creating and editing Mermaid diagrams. It is a **single self-contained HTML file** (`mermaid-forge.html`) with zero build steps — no package.json, no bundler, no backend. All logic lives in inline `<style>` and `<script>` tags within that file.

## Development Workflow

There is no build step. Open `mermaid-forge.html` directly in a browser to run the app. To test changes:
- Open the file locally in a browser (or serve it with `python3 -m http.server`)
- Test manually across Chrome (desktop/Android), Firefox, and Safari iOS — the primary test surface
- The 768px breakpoint switches from mobile (tab-based Code/Preview) to desktop (side-by-side split pane)

## Architecture

The entire application is structured as a single HTML file with logical sections:

### Data Layer
- **IndexedDB** (`"mermaid-forge"` database, `"diagrams"` store, keyed by `id`): Stores diagram records (`{id, name, code, createdAt, updatedAt, ghPath, ghSha}`). Accessed via helpers `openDB()`, `dbGetAll()`, `dbGet()`, `dbPut()`, `dbDelete()`.
- **localStorage**: Persists settings (AI provider/key, GitHub token/repo, mermaid theme, autoRender toggle) and the currently active diagram ID.

### External Integrations
- **Mermaid.js**: Loaded from CDN (`cdnjs.cloudflare.com`, fallback `jsdelivr.net`) at runtime via `loadMermaidScript()`. Rendered with `renderDiagram()` (debounced 800ms).
- **AI providers** (BYOK — bring your own key): Anthropic, OpenAI, and OpenRouter. Handled by `callAI()`. Keys are stored in localStorage and never leave the browser except in direct API calls to the respective provider.
- **GitHub API**: Fine-grained PAT with Contents R/W on a single repo. Functions `ghHeaders()`, `ghFilePath()`, `ghGetFile()`, `ghPutFile()`, `ghListFiles()` handle sync. Manual push/pull only — no auto-sync.

### UI Structure
- **Toolbar**: diagram name (editable), action buttons (files, export, sync, settings)
- **Tab bar** (mobile only): toggles Code/Preview panes
- **Editor pane**: plain textarea
- **Preview pane**: rendered SVG output with inline error display
- **AI prompt bar**: textarea + send button at bottom of screen
- **Modals**: diagram list and settings — slide-up from bottom, mobile-native feel

### Key Design Decisions
- No framework — vanilla JS/CSS only, intentionally minimal dependencies
- Dark theme with blue accent (`#3b82f6`), fonts DM Sans (UI) + JetBrains Mono (code editor)
- CSS custom properties for the design system; responsive breakpoint at 768px
- GitHub sync uses SHA-based optimistic locking (PUT requires `sha` from last GET to avoid clobber)

## Accessing GitHub PRs

`gh` is not available. Use the GitHub REST API directly with `curl`. The git remote username (`local_proxy`) also authenticates against the GitHub API:

```bash
# List open PRs
curl -s -u "local_proxy:" "https://api.github.com/repos/nverhaaren/mermaid-forge/pulls?state=open"

# PR review comments (the reviewer's overall summary)
curl -s -u "local_proxy:" "https://api.github.com/repos/nverhaaren/mermaid-forge/pulls/1/reviews"

# Inline code comments on a PR
curl -s -u "local_proxy:" "https://api.github.com/repos/nverhaaren/mermaid-forge/pulls/1/comments"

# General issue-style comments on a PR
curl -s -u "local_proxy:" "https://api.github.com/repos/nverhaaren/mermaid-forge/issues/1/comments"
```

Pipe through `python3 -m json.tool` for readable output, or filter with `python3 -c "import sys,json; ..."`.

## Reference Documents

- `mermaid-forge-overview.md` — project vision, target user, and competitive landscape analysis
- `mermaid-forge-phase1-2-plan.md` — detailed implementation roadmap for Phase 1 (local tool) and Phase 2 (PWA + GitHub sync)

## Phase Status

**Phase 1 (Local Tool)** — complete: IndexedDB, multi-diagram management, BYOK AI, import/export, auto-save, settings UI.

**Phase 2 (PWA + GitHub Sync)** — in progress: GitHub sync logic exists but needs edge-case hardening (401 token expiry messages, 409 SHA conflict detection). Still needed: `manifest.json`, `sw.js` service worker, app icons (`icons/icon-192.png`, `icons/icon-512.png`).
