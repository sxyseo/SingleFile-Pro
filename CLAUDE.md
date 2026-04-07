# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SingleFile is a browser extension (Web Extension, Manifest V2) that saves complete web pages as single HTML files. Compatible with Chrome, Firefox (Desktop/Android), Edge, Safari, Vivaldi, Brave, and Opera. Licensed under AGPL-3.0.

## Build Commands

```bash
# Install dependencies
npm install

# Development build (no minification)
npm run dev

# Production build (bundles + minifies to lib/, creates zip artifacts)
npm run build
# Note: build-extension.sh is a bash script requiring Linux/macOS (uses zip, jq, dpkg)

# Lint
npx eslint src/

# Bundle only (rollup production config)
npx rollup -c rollup.config.js
```

There are no tests in this repository.

## Architecture

### Dual-layer source structure

The extension has two distinct source layers that get bundled into `lib/`:

1. **`single-file-core` (npm dependency)** — The page-processing engine. Bundled from `node_modules/single-file-core/` into `lib/` files like `single-file.js`, `single-file-frames.js`, `single-file-bootstrap.js`, `single-file-hooks-frames.js`. This is the core HTML/CSS/resource extraction logic.

2. **`src/` (this repo)** — The extension-specific code: background scripts, content scripts, UI, and integration with cloud services. Also bundled into `lib/`.

Rollup (`rollup.config.js`) produces 18 bundles from both layers, outputting IIFE or UMD format files to `lib/`.

### Extension contexts (Manifest V2)

The extension runs code in three browser contexts:

- **Background page** (`src/core/bg/index.js` → `lib/single-file-extension-background.js`) — Message router. Dispatches incoming messages by prefix (`tabs.`, `downloads.`, `autosave.`, `ui.`, `config.`, `tabsData.`, `editor.`, `bookmarks.`, `companion.`, `requests.`, `bootstrap.`) to the appropriate module.

- **Content scripts** — Three injection groups defined in `manifest.json`:
  1. Frame scripts (all frames): `single-file-frames.js` + `single-file-extension-frames.js`
  2. MAIN world hooks (all frames): `single-file-hooks-frames.js`
  3. Top-frame bootstrap: `single-file-bootstrap.js` + `single-file-extension-bootstrap.js` + `single-file-infobar.js`

- **UI pages** (`src/ui/pages/`) — HTML pages for options, editor, panel, batch-save-urls, help, pendings, viewer.

### Key source directories

| Path | Purpose |
|---|---|
| `src/core/bg/` | Background modules: business logic (`business.js`), config (`config.js`), autosave, downloads, tabs management, external messages |
| `src/core/content/` | Content scripts: main save logic (`content.js`), bootstrap, frame handling |
| `src/core/common/` | Shared download utilities |
| `src/ui/bg/` | Background-side UI logic: menus, options, button state, commands, panel |
| `src/ui/content/` | Content-side UI: progress indicators, editor overlay |
| `src/lib/` | Integration libraries: Google Drive, Dropbox, GitHub, WebDAV, S3, Woleet (blockchain proof), MCP, YABSON (binary serialization), MHTML converter, Readability |
| `src/lib/single-file/` | Extension adapters for core: fetch hooks, frame-tree coordination, lazy-load timeouts, script injection |

### Message passing pattern

All cross-context communication uses `browser.runtime.sendMessage` / `browser.tabs.sendMessage` with a `{ method: "prefix.action" }` convention. The background page routes by prefix. Content scripts listen for `content.*` methods (`content.save`, `content.cancelSave`, `content.download`, etc.).

### Task queue

`src/core/bg/business.js` manages a task queue (`tasks[]`) for saving pages. Tasks have states (`pending` → `processing`) and respect `maxParallelWorkers`. Supports auto-save, batch URL saving, and cancellation.

## Code Style

- ES modules (`"type": "module"` in package.json)
- Double quotes, semicolons, unix line endings (enforced by ESLint)
- `/* global browser */` declarations for Web Extension APIs
- No TypeScript, no transpilation beyond Rollup bundling
- Each source file has the AGPL copyright header

## Key Files

- `manifest.json` — Extension manifest (MV2), content script injection config, permissions
- `rollup.config.js` — All 18 bundle definitions mapping source → `lib/` output
- `src/core/bg/config.js` — Default configuration and profile management
- `src/core/bg/business.js` — Core save task orchestration
- `src/core/content/content.js` — Page processing and download pipeline in content context
