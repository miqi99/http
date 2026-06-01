# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**http-client** is a Vue 3 + Vite web application for testing HTTP requests. It's a lightweight, browser-based alternative to Postman/Insomnia. The app uses localStorage for persistent state and supports GET/POST, custom headers, cookies, form parameters, JSON bodies, CURL import, and request history.

## Development Commands

- **`npm run dev`** - Start the Vite dev server. The app runs at http://localhost:5173 with HMR enabled
- **`npm run build`** - Build for production (output to `dist/`)
- **`npm run preview`** - Preview the built app locally

## Architecture

The application is a **single-file Vue component** (`src/App.vue`, ~1400 lines). State management is entirely in `<script setup>` with Vue ref() and computed() composables.

### Key State

All application state lives in `App.vue`:
- **Request config**: `url`, `method` (GET/POST), `accessMode` (direct/proxy), `paramType` (none/form/json)
- **Parameters**: `headers`, `cookie`, `formParams`, `jsonBody`
- **Response**: `response`, `responseInfo` (status, time, size, contentType, url), `loading`, `timeout`
- **Persistence**: `persistConfig` flag; two localStorage keys: `http-client-config` and `http-client-history`
- **Modals**: `showCurlModal`, `showHistoryModal` for UI overlays

### Request Sending

- **`sendRequest()`** - Main function that validates inputs, calls `fetch()`, handles timeouts (5s), and captures response metadata
- **Helper functions**:
  - `buildHeaders()` - Assembles header object; auto-sets Content-Type for JSON/form POST if user hasn't
  - `buildBody()` - Constructs request body (string for JSON, URLSearchParams for form)
  - `validateJsonBeforeSend()` - Validates JSON syntax before sending

### CURL Import

- **`importCurl()`** - Parses curl command strings using `splitCurlCommand()` to extract URL, method, headers, cookies, body
- **`parseFormString()`** - Converts `a=b&c=d` strings to form parameter rows

### Persistence & History

- **`onMounted()`** - Restores persisted config and history from localStorage on page load
- **`watch()`** - Monitors all config changes; if `persistConfig` is true, auto-saves to localStorage on every change
- **`saveHistoryEntry()`** - Appends request to history (max 20 recent entries); dedups by URL+method
- **`restoreHistory()`** - Fills current form from a history entry

### JSON Formatting

- **`displayRequestJson`** & **`displayResponse`** - Computed properties that format JSON when the user checks the "format" checkbox
- Gracefully falls back to raw text on parse error

## Styling

Uses **Tailwind CSS v4** for all styling. Configuration is in `vite.config.js`. Tailwind is imported and compiled via `@tailwindcss/vite` plugin.

## Notes for Future Development

- **Proxy mode** is stubbed out (line 719–732). When implemented, it will POST requests to `http://localhost:3000/proxy` to bypass CORS
- **HelloWorld.vue** is unused template boilerplate and can be deleted
- No tests; the app is small and UI-focused, making manual verification the norm
- All inputs are trusted (internal state only); no injection vectors to current API surface

# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
