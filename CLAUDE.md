# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Book Replicator is a **single-file browser application** (`index.html`). The entire app — HTML, CSS, and JavaScript — lives in one file. There is no build system, no package manager, and no test suite. To run it, open `index.html` directly in a browser or serve it with any static file server:

```
python3 -m http.server 8080
# or
npx serve .
```

## Architecture

The app calls the Anthropic API **directly from the browser** using the `anthropic-dangerous-direct-browser-access: true` header. The user supplies their own API key, which is persisted in `localStorage` under the key `book_replicator_api_key`.

### Three-step workflow

**Step 1 — Identify**: Accepts a cover image (base64-encoded, sent as a vision message) or a text description. Calls `claude-sonnet-4-6` with `IDENTIFY_PROMPT` and expects a JSON response with fields: `title`, `author`, `genre`, `year`, `description`, `confidence`, `notes`.

**Step 2 — Analyze**: Calls `claude-opus-4-6` with `ANALYZE_PROMPT` and returns a JSON object describing plot structure, themes, tone, POV, pacing, motifs, and comparable authors. The user can optionally paste book excerpts to improve accuracy. The analysis result is stored in `analysisData` and rendered as a card grid.

**Step 3 — Generate**: A two-phase process using `claude-opus-4-6`:
1. **Structure phase** (`buildStructurePrompt`): Non-streaming call that returns a JSON book blueprint — title, premise, setting, characters, and a chapter-by-chapter plan.
2. **Chapter phase** (`buildChapterPrompt`): Streams each chapter individually using server-sent events. Chapters are appended incrementally to `generatedText` and re-parsed with `marked` on each chunk.

User-controlled generation settings are tracked in the `fmt` object: `chapters` (5/10/15/20), `chapLen` (500–2000 words), `pov`, `genre`, `tone`, and `extra` (free-text instructions).

### Export

After generation, the user can export as:
- **DOCX** — via `html-docx-js` (CDN): markdown → HTML → `.docx` blob
- **PDF** — opens a new window and calls `window.print()`
- **EPUB** — via `JSZip` (CDN): splits `generatedText` on `## ` headings, writes EPUB 3.0 package structure into a zip

### Key implementation details

- All user-generated strings rendered into HTML go through `esc()` (manual HTML entity encoding) to prevent XSS.
- `stripJsonFences()` strips markdown code fences from API responses before `JSON.parse`.
- `callApi()` is the non-streaming wrapper; `streamApi()` is an async generator that parses SSE lines.
- `setLoading(btn, label)` / `resetLoading(btn, label)` manage button disabled state during async work.
- Step navigation (`goToStep(n)`) toggles `.active` CSS class and updates progress dot/line state.
- CDN dependencies: Tailwind CSS, `marked`, `html-docx-js@0.3.1`, `jszip@3`.

## Conventions

- The IIFE `(function () { 'use strict'; ... })()` wraps all JS — keep this pattern.
- No framework, no modules — all DOM refs are grabbed once at the top of the IIFE and reused.
- Prompts (`IDENTIFY_PROMPT`, `ANALYZE_PROMPT`, `buildStructurePrompt`, `buildChapterPrompt`) are the core logic. Changes to prompts affect the quality and schema of Claude's output; the JSON parsing downstream depends on exact field names.
- Model assignments are intentional: `claude-sonnet-4-6` for the fast identification call, `claude-opus-4-6` for quality-sensitive analysis and generation.
