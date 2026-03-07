# Mermaid Forge — Phase 1 & 2 Development Plan

## Phase 1: Functional Local Tool

### Goal
Take the working HTML prototype and turn it into a reliable, self-contained diagram editor that stores diagrams locally, supports AI generation via BYOK, and handles import/export. At the end of Phase 1, the tool is usable by opening the HTML file directly in a mobile browser.

### 1.1 Project Structure

The Phase 1 deliverable is a single HTML file (all CSS, JS inline) plus a few static assets. This keeps deployment trivial — no build step, no bundler, no node_modules.

```
mermaid-forge/
├── index.html          # The app (self-contained)
├── manifest.json       # Added in Phase 2
├── sw.js               # Added in Phase 2
├── icons/              # Added in Phase 2
│   ├── icon-192.png
│   └── icon-512.png
└── README.md
```

During Phase 1 development, you can simply open `index.html` in Chrome on your phone (via a local file URL or a simple local server). No build tooling needed.

### 1.2 IndexedDB Storage Layer

**What it does:** Stores all diagrams as structured objects in the browser's IndexedDB, providing persistent storage that survives page reloads, browser restarts, and (on installed PWAs) is resistant to cache eviction.

**Schema:**

```
Database: "mermaid-forge" (version 1)
Object Store: "diagrams"
  Key Path: "id" (string, e.g. "m3kf7x92a")
  Index: "updatedAt" (numeric timestamp, for sorting)

  Record shape:
  {
    id: string,           // Generated: Date.now().toString(36) + random suffix
    name: string,         // User-editable diagram name
    code: string,         // Mermaid syntax source code
    createdAt: number,    // Unix timestamp (ms)
    updatedAt: number,    // Unix timestamp (ms), updated on every save
    ghPath: string|null,  // GitHub file path if synced (e.g. "diagrams/auth-flow.mmd")
    ghSha: string|null    // GitHub blob SHA for optimistic concurrency on updates
  }
```

**Implementation notes:**

- The raw IndexedDB API is callback-based and verbose. The prototype wraps it in promise-based helper functions (`dbGet`, `dbPut`, `dbGetAll`, `dbDelete`). For Phase 1 this is sufficient. If the wrapper becomes unwieldy, consider switching to the `idb` library (by Jake Archibald), which is ~1.2KB gzipped and provides a clean async/await interface.
- Auto-save triggers on a debounced `input` event from the editor textarea (800ms delay). This writes the current diagram's code and name to IndexedDB without user intervention.
- The `updatedAt` index allows the diagram list to be sorted by recency.

**Tasks:**

- [x] IndexedDB open/upgrade with diagrams object store
- [x] CRUD helpers: dbGet, dbPut, dbGetAll, dbDelete
- [x] Auto-save on editor input (debounced)
- [x] Diagram list modal with create/switch/delete
- [x] Persist current diagram ID in localStorage for reload continuity
- [ ] Migrate any existing `mf_code` localStorage data into IndexedDB on first boot (for users upgrading from the v1 prototype)

### 1.3 Multi-Diagram Management

**What it does:** A modal UI listing all saved diagrams with the ability to create new ones, switch between them, rename them, and delete them.

**UX flow:**

1. Tap the file icon in the toolbar → diagram list modal slides up from bottom
2. Current diagram is highlighted; each entry shows name, time-ago, and sync status
3. "New Diagram" button creates a blank diagram and switches to it
4. Tapping a diagram saves the current one, then loads the selected one
5. Delete button on each item (with confirmation) removes it from IndexedDB
6. Diagram name is editable directly in the toolbar (inline text input)

**Tasks:**

- [x] Diagram list modal with slide-up animation
- [x] Render list sorted by updatedAt descending
- [x] Create new diagram flow
- [x] Switch diagram (save current → load selected)
- [x] Delete with confirmation
- [x] Editable name in toolbar
- [ ] Handle edge case: deleting the last diagram (auto-create a new blank one)
- [ ] Truncate long names gracefully in the list

### 1.4 BYOK AI Integration

**What it does:** Sends the user's natural language prompt (plus optional existing diagram code as context) to their configured LLM provider, receives mermaid code back, and inserts it into the editor.

**Supported providers:**

| Provider | API Endpoint | Auth Header | Notes |
|----------|-------------|-------------|-------|
| Anthropic | `api.anthropic.com/v1/messages` | `x-api-key` | Requires `anthropic-dangerous-direct-browser-access: true` header for CORS. User must enable this in their Anthropic console. |
| OpenAI | `api.openai.com/v1/chat/completions` | `Authorization: Bearer` | Standard CORS support. |
| OpenRouter | `openrouter.ai/api/v1/chat/completions` | `Authorization: Bearer` | OpenAI-compatible API. Best option for browser-based BYOK since it's designed for this use case and supports many models. |

**System prompt:** Instructs the model to output only raw mermaid code with no markdown fences, explanations, or commentary. If existing code is provided, the model outputs the full modified diagram.

**Response cleaning:** Strips any markdown fences (` ```mermaid ``` `) that models sometimes include despite instructions.

**Tasks:**

- [x] Provider selection and model picker in settings
- [x] API key storage in localStorage
- [x] Prompt bar with send button and Enter-to-send
- [x] Loading indicator during generation
- [x] Response cleaning (strip fences)
- [ ] Error handling: surface specific API errors (401 = bad key, 429 = rate limit, network failure)
- [ ] Consider adding a "retry" option on failure

### 1.5 Import and Export

**Export SVG:** Serializes the rendered SVG element from the DOM into a Blob and triggers a download. Filename is based on the diagram name.

**Export .mmd:** Saves the raw mermaid source code as a plain text file with `.mmd` extension. This is the portable format — it can be opened in mermaid.live, rendered by GitHub in markdown files, or processed by any mermaid-compatible tool.

**Import .mmd:** File picker accepting `.mmd`, `.mermaid`, `.md`, and `.txt` files. Creates a new diagram in IndexedDB with the file contents and the filename (minus extension) as the diagram name.

**Tasks:**

- [x] Export dropdown (SVG / .mmd)
- [x] SVG export via Blob + download
- [x] .mmd export via Blob + download
- [x] Import via hidden file input
- [ ] Consider drag-and-drop import on the editor area (nice to have)

### 1.6 Templates

**What it does:** Pre-built starter diagrams for common diagram types, accessible from the toolbar. Loads the template code into a new diagram.

**Included templates:** Flowchart, Sequence Diagram, Class Diagram, ER Diagram, State Diagram, Gantt Chart, Git Graph.

**Tasks:**

- [x] Template data (code strings for each type)
- [x] Template picker dropdown
- [ ] Consider: should loading a template create a new diagram or replace the current editor content? Current behavior: replaces content in the current diagram. May want to offer a choice.

### 1.7 Testing (Phase 1)

Since this is a single-file browser app with no build step, testing is primarily manual and device-focused.

**Manual test matrix:**

| Test | Chrome Android | Chrome Desktop | Safari iOS | Firefox |
|------|---------------|----------------|------------|---------|
| Create diagram, type code, see preview | | | | |
| Switch between Code/Preview tabs (mobile) | | | | |
| Side-by-side layout (desktop/tablet) | | | | |
| Create multiple diagrams, switch between them | | | | |
| Delete a diagram, verify list updates | | | | |
| Rename a diagram via toolbar input | | | | |
| Close and reopen browser — diagrams persist | | | | |
| AI generation (at least one provider) | | | | |
| Export SVG — verify valid SVG file | | | | |
| Export .mmd — verify correct content | | | | |
| Import .mmd — verify new diagram created | | | | |
| Load each template, verify it renders | | | | |
| Type invalid mermaid syntax — verify error shown | | | | |
| Settings: change theme, verify diagram re-renders | | | | |

**Key things to verify on mobile specifically:**

- Toolbar buttons don't overlap on narrow screens (< 360px width)
- Diagram name input is usable (not too narrow)
- Bottom prompt bar doesn't get hidden behind the virtual keyboard
- Tab switching is responsive
- Modals are scrollable when content exceeds viewport height

---

## Phase 2: PWA + GitHub Sync

### Goal
Make the app installable as a standalone PWA with offline support, and add GitHub sync for cross-device diagram portability. At the end of Phase 2, the app can be added to an Android home screen, works offline, and syncs diagrams to/from a GitHub repository.

### 2.1 Web App Manifest

**What it does:** Tells the browser that this web page is an installable application. Chrome on Android will show an "Add to Home Screen" prompt (or the user can manually add it from the browser menu).

**File: `manifest.json`**

```json
{
  "name": "Mermaid Forge",
  "short_name": "MermaidForge",
  "description": "Mobile-first mermaid diagram editor with BYOK AI",
  "start_url": "/index.html",
  "display": "standalone",
  "background_color": "#0a0e17",
  "theme_color": "#0a0e17",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**Key fields:**

- `display: standalone` — launches without browser chrome (no URL bar)
- `theme_color` — colors the Android status bar
- `start_url` — what loads when the app is launched from the home screen
- Icons must be at least 192px and 512px for Chrome's installability criteria

**Link from HTML:**

```html
<link rel="manifest" href="manifest.json">
```

**Tasks:**

- [ ] Create manifest.json
- [ ] Generate app icons (192px and 512px PNG)
- [ ] Add manifest link to index.html
- [ ] Add `<meta name="theme-color" content="#0a0e17">` to HTML head
- [ ] Test: verify Chrome shows install prompt or "Add to Home Screen" option
- [ ] Test: verify app launches in standalone mode (no URL bar)

### 2.2 Service Worker

**What it does:** A JavaScript file that runs in a separate thread from the page, intercepting all network requests. On first load, it pre-caches the app's static assets. On subsequent loads, it serves them from cache, enabling offline use.

**File: `sw.js`**

```javascript
const CACHE_NAME = 'mermaid-forge-v1';
const ASSETS = [
  '/',
  '/index.html',
  'https://cdnjs.cloudflare.com/ajax/libs/mermaid/11.4.1/mermaid.min.js',
  'https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600&family=DM+Sans:wght@400;500;600;700&display=swap',
];

// Install: pre-cache all static assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS))
  );
  self.skipWaiting(); // Activate immediately
});

// Activate: clean up old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((k) => k !== CACHE_NAME).map((k) => caches.delete(k)))
    )
  );
  self.clients.claim(); // Take control of all pages immediately
});

// Fetch: cache-first for static assets, network-first for API calls
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // API calls (AI providers, GitHub) should always go to network
  if (url.hostname.includes('api.anthropic.com') ||
      url.hostname.includes('api.openai.com') ||
      url.hostname.includes('openrouter.ai') ||
      url.hostname.includes('api.github.com')) {
    return; // Don't intercept — let the browser handle normally
  }

  // Everything else: try cache first, fall back to network
  event.respondWith(
    caches.match(event.request).then((cached) => cached || fetch(event.request))
  );
});
```

**Lifecycle to understand:**

1. Browser downloads `sw.js` and runs the `install` event → assets get cached
2. On activation, old caches are purged
3. On every fetch, static assets are served from cache; API calls pass through to the network
4. To push an update: change `CACHE_NAME` (e.g. `v1` → `v2`) and the list of assets. The browser detects the changed `sw.js`, installs the new version, and on next page load the old cache is cleaned up.

**Register from HTML:**

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

**Tasks:**

- [ ] Create sw.js with install/activate/fetch handlers
- [ ] Add service worker registration to index.html
- [ ] Include mermaid.js CDN URL in pre-cache list
- [ ] Include Google Fonts CSS URL in pre-cache list (note: font files themselves are fetched by the CSS and will be cached on first use via the cache-first fetch handler)
- [ ] Test: load app, go offline (airplane mode), verify app still works
- [ ] Test: verify diagram editing and rendering works offline
- [ ] Test: verify AI features show appropriate error when offline (not a crash)
- [ ] Test: update CACHE_NAME, redeploy, verify old cache is purged on next visit

### 2.3 Hosting

The app needs to be served over HTTPS for service workers to function. Options:

**GitHub Pages (recommended for starting):**

- Free, automatic HTTPS, deploys from a branch or `/docs` folder
- Push the repo, enable Pages in settings, done
- Custom domain support if desired later
- URL: `https://username.github.io/mermaid-forge/`

**Cloudflare Pages (alternative):**

- Slightly faster global CDN
- Also free for static sites
- Connects to a GitHub repo for auto-deploy

**Setup tasks:**

- [ ] Create a GitHub repo for the app source (separate from the diagrams repo)
- [ ] Push index.html, manifest.json, sw.js, icons/
- [ ] Enable GitHub Pages (Settings → Pages → Source: main branch)
- [ ] Verify the app loads at the Pages URL
- [ ] Verify service worker registers and caching works

**Important:** The app source repo and the diagrams sync repo should be separate. The app repo holds the tool's code. The diagrams repo holds the user's `.mmd` files. They serve different purposes and have different access patterns.

### 2.4 GitHub Sync

**What it does:** Provides manual push (upload current diagram to GitHub) and pull (download all diagrams from GitHub) operations using the GitHub Contents API. Diagrams are stored as `.mmd` text files in a dedicated repository.

**Authentication: Fine-Grained Personal Access Token**

The app guides the user through creating a token with the absolute minimum permissions:

1. Go to GitHub → Settings → Developer settings → Fine-grained personal access tokens → Generate new token
2. Token name: `mermaid-forge`
3. Expiration: user's choice (recommend 90 days; they'll need to regenerate and re-enter periodically)
4. Repository access: **Only select repositories** → pick the diagrams repo
5. Permissions → Repository permissions → **Contents: Read and write**
6. All other permissions: No access
7. Generate and paste into the app's settings

This grants: read and write files in one repo. It does not grant access to issues, PRs, settings, other repos, or any org-level permissions.

**Push flow (single diagram):**

```
1. Build file path: {prefix}{sanitized_name}.mmd
2. GET /repos/{owner}/{repo}/contents/{path}
   → 404: file doesn't exist yet (new)
   → 200: file exists, extract `sha` for update
3. PUT /repos/{owner}/{repo}/contents/{path}
   Body: { message, content (base64), sha (if updating) }
   → 200/201: success, extract new sha from response
4. Store ghPath and ghSha on the diagram in IndexedDB
```

**Pull flow (all diagrams):**

```
1. GET /repos/{owner}/{repo}/contents/{prefix}
   → List of files, filter for *.mmd
2. For each .mmd file:
   a. GET the file to retrieve content (base64) and sha
   b. Decode content from base64
   c. Check if a local diagram already has this ghPath
      → Yes: update its code and ghSha
      → No: create a new diagram with the file's name and content
3. Reload the diagram list
```

**Edge cases and limitations:**

- **Name collisions:** If two local diagrams have the same name, they'd map to the same GitHub path. The sanitization function should handle this (or the user renames one).
- **Conflict resolution:** This is a last-write-wins model. If the same diagram is edited on two devices and pushed from both, the second push will fail (SHA mismatch). The user would need to pull first, which overwrites local changes. For a single-user tool this is rarely an issue, but worth documenting.
- **File path encoding:** GitHub API paths need proper encoding for special characters. The sanitization function strips non-alphanumeric characters except underscores, hyphens, dots, and spaces.
- **Rate limiting:** GitHub API allows 5,000 requests/hour for authenticated users. A pull of 50 diagrams uses ~51 requests (1 list + 50 file fetches). Not a concern for normal use.
- **Token expiration:** Fine-grained PATs expire. The app should surface a clear error when the token is expired (401 from GitHub) and direct the user to settings to update it.

**Tasks:**

- [x] GitHub API helpers (ghGetFile, ghPutFile, ghListFiles)
- [x] Push current diagram button
- [x] Pull all diagrams button
- [x] Store ghPath and ghSha on diagram records
- [x] Settings UI for token, repo, path prefix
- [x] Guided PAT creation instructions
- [x] Sync status indicator per diagram (local / synced)
- [ ] Handle 401 errors with clear "token expired or invalid" message
- [ ] Handle SHA conflict on push (409) with guidance to pull first
- [ ] Add a "push all" option (push every diagram that has local changes)
- [ ] Consider: track local modification state (compare code hash to last-pushed version) to show "modified" vs "synced" status more accurately

### 2.5 Testing (Phase 2)

**PWA install and offline tests:**

| Test | Steps | Expected |
|------|-------|----------|
| Install prompt | Visit site in Chrome Android → look for install banner or use menu → "Add to Home Screen" | App icon appears on home screen |
| Standalone launch | Tap home screen icon | App opens without URL bar, status bar matches theme color |
| Offline editing | Launch app → enable airplane mode → create/edit diagram | Editing and rendering work normally |
| Offline AI | Attempt AI generation while offline | Clear error message, no crash |
| Cache update | Change CACHE_NAME in sw.js, redeploy → revisit app | New version loads on second visit |
| Storage persistence | Install PWA → create diagrams → force-stop Chrome → reopen app | All diagrams still present |

**GitHub sync tests:**

| Test | Steps | Expected |
|------|-------|----------|
| First push | Configure token + repo → create diagram → push | File appears in GitHub repo |
| Update push | Edit a synced diagram → push again | File updated in repo (new commit) |
| Pull to new device | Open app in a different browser → configure same token/repo → pull | All diagrams downloaded and accessible |
| Push with expired token | Set an expired/invalid token → attempt push | Clear error message about authentication |
| Pull empty repo | Configure a repo with no .mmd files → pull | "No .mmd files found" message |
| Rename and push | Rename a previously-synced diagram → push | New file created with new name (old file remains — document this as expected behavior) |

**Cross-browser verification:**

| Browser | PWA Install | Offline | IndexedDB | GitHub Sync |
|---------|-------------|---------|-----------|-------------|
| Chrome Android | Full support | Yes | Yes | Yes |
| Chrome Desktop | Yes | Yes | Yes | Yes |
| Safari iOS | Add to Home Screen (partial PWA) | Yes | Yes | Yes |
| Firefox Android | No install prompt | Yes | Yes | Yes |
| Samsung Internet | Yes | Yes | Yes | Yes |

Note: Safari on iOS has some PWA limitations (no push notifications, limited background sync, potential IndexedDB eviction after 7 days of non-use for non-home-screen apps). For our use case (no push notifications needed, sync is manual), this is acceptable. Adding to Home Screen on iOS prevents the 7-day eviction.

### 2.6 Deployment Checklist

Before considering Phase 2 complete:

- [ ] App loads and functions at the hosted URL
- [ ] Service worker registers and caches assets on first visit
- [ ] App works fully offline after first visit (except AI and sync features)
- [ ] "Add to Home Screen" works on Chrome Android
- [ ] App launches in standalone mode from home screen
- [ ] At least one full push/pull cycle tested with a real GitHub repo and token
- [ ] Settings clearly explain token creation with minimal permissions
- [ ] All API keys and tokens stored only in localStorage, never transmitted to any server other than the configured provider
- [ ] README.md updated with setup instructions
