# AGENTS.md — AI Context Studio

## Project Overview

Single-page HTML app (`index.html`) that exports project files as AI-ready context. Users drag-drop a project folder, select files via a tree UI, and copy/download the output in TXT/MD/XML. No build step, no dependencies — one self-contained file.

## Architecture

```
index.html
├── <style>   — All CSS (dark theme, custom checkboxes, layout)
├── <body>    — Two-panel layout: left (tree + controls), right (preview + export)
└── <script>  — All JS (~1000 lines, structured in labeled sections)
```

### Key Sections in `<script>` (in order)

| Section | Purpose |
|---------|---------|
| ICONS / Config | SVG icon strings, binary extensions, always-ignored dirs |
| State | Single `state` object — source of truth for everything |
| Helpers | `$()`, `toast()`, `debounce()`, `copyToClipboard()`, `formatSize()` |
| Gitignore Parser | Full `.gitignore` spec: `**`, `!`, `?`, negation, anchoring |
| Custom Ignore | Glob matching with regex cache, checks dir segments too |
| Undo System | Snapshot-based undo stack (max 30 entries) |
| File Processing | `processFileSystem(items, append)` — walks FileSystem/FileList |
| Visibility | `getVisibleFiles()` — cached per render cycle |
| Folder Stats | `computeFolderStats()` — pre-computed O(n) for tree checkboxes |
| Tree Rendering | `buildTree()` / `buildFolder()` / `buildFile()` — DOM construction |
| Stats / FS Gen | Token estimation, file structure text generation |
| State Gen/Parse | Selection state textarea — bidirectional sync |
| Export | Lazy content generation — reads files only on copy/download |
| Events | All event bindings, keyboard shortcuts |

### Rendering Pipeline

```
renderApp()
  → _cachedVisible = null        // invalidate cache
  → getVisibleFiles()             // filter + cache result
  → computeFolderStats(visible)   // O(n) single pass for all folder checkbox states
  → buildTree()                   // full DOM rebuild from cached visible files
  → updateStats()                 // token count + progress bar
  → generateFS()                  // file structure text
  → generateStateText()           // selection state textarea (skipped if focused)
```

**Critical**: `getVisibleFiles()` is cached per render cycle. The cache is invalidated at the start of `renderApp()`. Any code that mutates file state must call `renderApp()` to see changes.

### State Shape

```js
state.files[]         // { path, name, size, fileRef, isSelected, isExcluded, isGitignored }
state.gitignoreMatchers[]    // parsed .gitignore rules
state.customIgnorePatterns[] // user-added glob patterns
state.useGitignore           // toggle
state.searchQuery            // lowercase search string
state.format                 // "txt" | "md" | "xml"
state.fsSelectedOnly         // show only selected in structure preview
state.fsShowTokens           // show token counts in file structure output
state.expandedPaths          // Set<string> for tree open/close persistence
state.undoStack[]            // { description, snapshot[] }
state._folderStats           // Map<path/, {total, selected}> — computed per render
```

## Patterns & Conventions

- **No framework** — vanilla JS, DOM API only.
- **Single source of truth** — the `state` object. UI is always derived from state via `renderApp()`.
- **Immutable-ish renders** — `renderApp()` rebuilds the entire tree. No partial DOM patching.
- **Lazy file reading** — `fileRef.text()` is only called on copy/download, never during rendering.
- **Debounced search** — 120ms debounce on search input to avoid render storms.
- **Undo via snapshots** — each undoable action saves `{path, isSelected, isExcluded}` for all files.
- **Custom ignore regex cache** — `_ignoreRegexCache` Map avoids recompiling patterns every filter pass.

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Ctrl/Cmd + A` | Select/deselect all visible files |
| `Ctrl/Cmd + Z` | Undo last action |
| `Escape` | Clear search input |

## Code Style

- 4-space indent (HTML + CSS + JS).
- `camelCase` for variables/functions, `UPPER_SNAKE` for constants.
- Template literals for HTML injection, `document.createElement` for interactive elements.
- CSS custom properties (`--var`) for theming.
- No semicolons are fine — the codebase uses them, keep consistent.

## Common Tasks

**Add a new export format**: Add a `tab-btn` in HTML, handle the format string in `generateExportContent()`.

**Add a new always-ignored directory**: Add to `ALWAYS_IGNORED_DIRS` Set in Config section.

**Add a new binary extension**: Add to `BINARY_EXTS` Set in Config section.

**Change token limit**: Update `TOKEN_LIMIT` constant and the `/1M` label in HTML.

## Versioning

Version is displayed in the header UI. Update both the `<span>` in HTML and commit message when bumping. Current: `v0.4.2`.

**IMPORTANT**: Always bump the version when adding features or making changes. Never forget this step. Patch bump (0.x.Y) for small features/fixes, minor bump (0.X.0) for significant features.
