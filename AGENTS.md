# CLAUDE.md — margin-exporter

## Project Overview

**margin-exporter** is a client-side web tool for exporting annotations, highlights, and bookmarks from [margin.at](https://margin.at) using the AT Protocol (the open, distributed protocol powering Bluesky). It was built without any frameworks or build tools.

The entire application runs in the browser — credentials never leave the user's device.

---

## Repository Structure

```
margin-exporter/
├── index.html      # Entire application: HTML, CSS, and JavaScript in one file (~1,380 lines)
├── about.html      # User-facing documentation/tutorial page (~560 lines)
├── favicon.svg     # Site icon
├── README.md       # Brief project description and TODO list
└── .gitattributes  # Git line-ending normalization
```

There are **no build tools, no package manager, no dependencies, and no tests**.

---

## Technology Stack

- **Vanilla JavaScript** (ES6+) — no frameworks, no bundlers
- **HTML5 + CSS3** — single-file app with inline styles and scripts
- **AT Protocol (XRPC)** — for authenticating with and querying the user's Personal Data Server (PDS)
- **Browser Fetch API** — all HTTP requests
- **Browser localStorage** — credential caching only
- **Google Fonts** — Lora, DM Sans, DM Mono (loaded from CDN)

---

## Application Architecture

### Single-File Design

All application logic lives in `index.html`. It is organized into clear sections:

| Lines (approx) | Section |
|---|---|
| 1–9 | HTML head, meta, font imports |
| 10–178 | CSS (variables, layout, components, responsive) |
| 179–342 | HTML structure (header, sidebar, main panel, modals) |
| 343–1376 | JavaScript (all logic) |

### JavaScript Structure (`index.html`)

**State Variables** (lines ~345–373):
```javascript
var session = null            // AT Protocol session
var pdsBase = 'https://bsky.social'  // Resolved PDS URL
var allCollections = []       // margin.at collections
var activeCollection = null   // Currently selected collection
var activeAnnotations = []    // Annotations in active collection
var browseAllRecords = null   // All annotations for browse-all view
var browseSelectedUris = new Set()  // Selected URIs for export
```

**Functional Groups:**
- **Auth & Credentials** — `login()`, `resolvePds()`, `onLoggedIn()`, `logout()`, `saveCreds()`, `loadCreds()`, `clearCreds()`
- **AT Protocol Helpers** — `atRequest()`, `listRecordsAll()`, `getRecord()`, `parseAtUri()`, `uriFrom()`
- **Collections & Data Loading** — `loadCollections()`, `selectCollection()`, `fetchAnnotationsForCollection()`
- **UI Rendering** — `renderAnnotations()`, `renderCards()`, `cardHTML()`, `renderLoading()`, `renderError()`
- **Collection Export** — `openExportModal()`, `doExport()`, `openTagExportModal()`, `doTagExport()`
- **Browse All** — `openBrowseAll()`, `renderBrowseAll()`, `renderBrowseCards()`, `getBrowseFiltered()`
- **Selection Export** — `openSelectionExportModal()`, `doSelectionExport()`
- **Full Account Backup** — `exportAllAccount()`
- **Export Format Builders** — `buildMarkdown()`, `buildCleanMarkdown()`, `csvCell()`, `download()`
- **Field Extraction & Utilities** — `extractFields()`, `annType()`, `formatDate()`, `escHtml()`, `truncate()`

---

## AT Protocol Integration

### Authentication Flow

1. User provides AT Protocol **handle** (e.g. `user.bsky.social`) and an **app password**
2. `resolvePds(handle)` discovers the user's PDS:
   - Tries `https://<handle>/.well-known/atproto-did` (custom domain DIDs)
   - Falls back to `https://bsky.social/xrpc/com.atproto.identity.resolveHandle`
   - Fetches DID document from `plc.directory` (did:plc) or the handle domain (did:web)
   - Extracts the PDS URL from the DID document's `#atproto_pds` service entry
3. `com.atproto.server.createSession` authenticates and returns `accessJwt`

### XRPC Endpoints Used

| Method | Type | Purpose |
|---|---|---|
| `com.atproto.identity.resolveHandle` | GET | Resolve handle → DID |
| `com.atproto.server.createSession` | POST | Authenticate |
| `com.atproto.repo.listRecords` | GET | Paginated record listing |
| `com.atproto.repo.getRecord` | GET | Fetch individual record |

### margin.at Record Types (NSIDs)

```javascript
const NSID = {
  collection:     'at.margin.collection',
  collectionItem: 'at.margin.collectionItem',
  annotation:     'at.margin.annotation',
  highlight:      'at.margin.highlight',
  bookmark:       'at.margin.bookmark',
  reply:          'at.margin.reply'
}
```

### Record Schemas

**Annotation:** `target.source` (URL), `target.title`, `target.selector.exact` (quote), `body.value` (note)

**Highlight:** `uri`/`url` (URL), `title`, `exact`/`text`/`quote` (quote), `comment`/`note` (note)

**Bookmark:** `source` (URL), `title`

The `extractFields(rec)` function normalizes these different schemas into a unified shape for rendering and export.

---

## Export Formats

| Format | Function | Output |
|---|---|---|
| JSON | `doExport('json')` | Raw AT Protocol record objects |
| CSV | `doExport('csv')` | Spreadsheet with all fields |
| Markdown | `buildMarkdown()` | Full markdown with metadata |
| Beautified Markdown | `buildCleanMarkdown()` | Clean quotes + notes only |

---

## Code Conventions

### JavaScript

- **`var`** declarations — not `const`/`let` (project style)
- **Async/await** for all asynchronous operations
- **String concatenation** for HTML generation (no template engines)
- **Direct DOM manipulation** via `innerHTML` and property assignment
- IIFEs for isolated one-time operations: `(function prefill() { ... })()`
- Section headers use: `// ── Section Name ──` style

### Naming

- `camelCase` — functions and variables
- `SCREAMING_SNAKE_CASE` — constants (`NSID`, `STORAGE_KEY`)
- Function prefixes: `open*`, `close*`, `do*`, `render*`, `set*`, `build*`
- HTML IDs: prefixed by feature area (`login-*`, `export-*`, `browse-*`)

### CSS

- **CSS custom properties** for theming (defined in `:root`)
- **BEM-like** class naming: `.card-header`, `.modal-footer`, `.state-box`
- **Flexbox and Grid** for layout
- **Mobile-first** responsive with `@media (max-width: 600px)`

### HTML

- Semantic elements: `<header>`, `<aside>`, `<main>`, `<section>`
- `data-*` attributes for dynamic behavior: `data-uri`, `data-type`, `data-tag`
- `for`/`id` pairing on all form labels

---

## Development Workflow

### Running Locally

No build step needed. Open `index.html` directly in a browser:

```bash
open index.html
# or
python3 -m http.server 8080  # then visit http://localhost:8080
```

### Making Changes

1. Edit `index.html` (and/or `about.html`) directly
2. Refresh the browser to see changes
3. No compilation, no bundling, no hot reload setup required

### Testing

There is no automated test suite. All testing is manual:
- Log in with a real margin.at AT Protocol account
- Verify collections load, annotations render, filters work
- Test each export format (JSON, CSV, Markdown, Beautified Markdown)
- Test the full account backup export
- Test Browse All view with type, tag, and date filters

### Git Branch Convention

Feature branches follow the pattern: `claude/<description>-<id>`

---

## Security Model

- **Client-side only** — no server, no backend, no analytics
- **Credentials stored in localStorage** under key `margin_at_creds` — can be cleared via "Forget credentials" button
- **App passwords** (not main account passwords) are used — scoped and revocable from Bluesky settings
- **No third-party requests** except Google Fonts and the user's own AT Protocol PDS

---

## Known TODOs (from README.md)

- [ ] Fix login "Continue" button not updating when fields are completed
- [ ] Add a web page explaining what the tool does and how to use it (about.html partially addresses this)

---

## Key Files Reference

| File | Purpose |
|---|---|
| `index.html` | Entire application |
| `about.html` | User documentation / how-to guide |
| `favicon.svg` | SVG site icon |
| `README.md` | Brief description + TODOs |
