# Harness — a local-first learning workspace

Harness is a mobile-first learning workspace built around **Leo**, an AI tutor that works *with* your own materials instead of replacing them. You create a project for anything you want to learn, drop in PDFs, audio, video, images and notes, and chat with Leo. Leo places you at the right level, teaches, quizzes you, and keeps a living map of what you know.

Everything lives **on your device**. There is no backend and no account. Your projects, files, notes, chats and memories are stored in the browser's IndexedDB. The only network traffic is to the AI provider *you* configure (or none at all if you use a local model such as Ollama).

> **Status:** early-stage / personal project. The app ships as a single self-contained `index.html`, so it can be hosted anywhere static files are served, or opened as a PWA on your phone.

---

## Table of contents

- [Highlights](#highlights)
- [How it works](#how-it-works)
- [Getting started](#getting-started)
- [Connecting an AI provider](#connecting-an-ai-provider)
- [Using Harness](#using-harness)
- [Leo's action protocol](#leos-action-protocol)
- [Privacy and data](#privacy-and-data)
- [Backups](#backups)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Development](#development)
- [Visual QA tooling](#visual-qa-tooling)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Highlights

- **Local-first.** All data is stored in IndexedDB via Dexie. Works offline except for AI calls and the in-app page reader.
- **Bring your own model.** OpenAI, Anthropic, Google Gemini, OpenRouter, NVIDIA NIM, Groq, Ollama, LM Studio, or any OpenAI-compatible endpoint.
- **Adaptive placement.** A short conversation finds your level and starting point for a topic.
- **Teach and test.** Leo teaches small chunks, then checks understanding with graded quizzes that update your knowledge map.
- **Knowledge map and spaced review.** Concepts are tracked as *not started, learning, shaky* or *solid*, with an automatic review schedule.
- **Grounded in your library.** PDFs and notes are indexed on-device (BM25). Leo cites passages like `book.pdf, p.12`.
- **Point at exactly what you mean.** `@`-mention a specific PDF page, image, or an audio segment, or just select text in a note or PDF and ask.
- **Real tools, not just text.** Leo can create notes, draw Excalidraw sketches, update your map, save memories and open reference pages.
- **Long-term memory you control.** Saved learner facts are visible, editable and deletable.
- **Backup and restore.** Export everything, including files, to a single JSON file.
- **Single-file build.** `vite build` produces one `index.html` with all JS, CSS and fonts inlined.

---

## How it works

```
┌──────────────────────────── Browser (PWA) ────────────────────────────┐
│                                                                       │
│  React 19 UI  ──►  Zustand (UI/context state)                         │
│       │                                                               │
│       ▼                                                               │
│  Dexie / IndexedDB  ◄── projects, files (Blobs), notes, sketches,     │
│       │                  threads, messages, concepts, points,         │
│       │                  memories, chunks, providers, kv              │
│       ▼                                                               │
│  Leo: system prompt  ──►  your chosen LLM provider  (streaming)       │
│       ▲                                   │                           │
│       │                                   ▼                           │
│  Action parser/validator  ◄──  ```action {json}``` blocks in reply    │
│       │                                                               │
│       └─► executes locally: map, points, notes, sketches, memory…     │
└───────────────────────────────────────────────────────────────────────┘
```

For each message, Harness builds a system prompt containing the project's topic, learner profile, knowledge map, points card, concepts due for review, relevant saved memories, and a list of library items. Leo does **not** receive your files wholesale. It only sees:

1. passages retrieved for the current message (shown as `SOURCES`),
2. whatever you explicitly attach (open note, selected text, current PDF page, image, sketch, `@`-mentions, captured web page).

Leo replies in Markdown and may include fenced `action` blocks containing JSON. The app validates each block against a strict schema, executes it locally, and shows you a result chip. Malformed blocks are shown to you and reported back to the model so it can correct itself.

---

## Getting started

### Prerequisites

- Node.js 20+ (developed on Node 22)
- npm

### Install and run

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
npm install
npm run dev
```

Open the URL Vite prints (usually `http://localhost:5173`).

### Production build

```bash
npm run build      # outputs dist/index.html (single self-contained file)
npm run preview    # serve the build locally
```

Because the build is one file, you can host `dist/index.html` on GitHub Pages, Netlify, Cloudflare Pages or any static host. For use as a phone app, open the hosted URL and choose **Add to Home Screen**.

> **Note:** AI providers must be reachable from the browser. HTTPS pages can only call HTTPS endpoints. For local models, see the [Ollama / LM Studio](#local-models) notes below.

### Try it without an API key

Turn on **Settings → Developer mode** to enable a built-in mock tutor ("Developer mock"). It streams scripted responses through the same action parser and executor as a real model, which is useful for exploring the UI and for testing.

---

## Connecting an AI provider

Go to **Settings → Add provider** and pick a preset (or *Custom*). Then enter an API key, optionally fetch the provider's model list, and enable the models you want.

| Preset | API type | Default base URL |
| --- | --- | --- |
| OpenAI | OpenAI-compatible | `https://api.openai.com/v1` |
| Anthropic | Anthropic | `https://api.anthropic.com` |
| Google Gemini | Gemini | `https://generativelanguage.googleapis.com` |
| OpenRouter | OpenAI-compatible | `https://openrouter.ai/api/v1` |
| NVIDIA NIM | OpenAI-compatible | `https://integrate.api.nvidia.com/v1` |
| Groq | OpenAI-compatible | `https://api.groq.com/openai/v1` |
| Ollama (local) | OpenAI-compatible | `http://localhost:11434/v1` |
| LM Studio (local) | OpenAI-compatible | `http://localhost:1234/v1` |
| Custom | OpenAI-compatible | *(you provide)* |

Per-provider options: request timeout (default 120 s), max output tokens (required by Anthropic), enabled models, and a default model. You can also choose a different provider/model **per chat thread**.

### Local models

- **Ollama:** start it with `OLLAMA_ORIGINS=*` so the browser may call it. From a phone, replace `localhost` with your computer's LAN IP.
- **LM Studio:** enable CORS in the server settings.

> **About API keys:** keys are stored in IndexedDB on your device and sent only to the provider you configured. Anthropic requests are made directly from the browser using Anthropic's browser-access header. Treat this like any client-side app: use keys with appropriate spending limits, and don't use the app on a device you don't trust.

---

## Using Harness

### Projects
Create a project per topic (for example *Linear algebra*, *Japanese*, *Rust*). Each project gets a name, short tag, color, and an emoji or uploaded image as its icon. Leo's avatar can be customized globally and per project.

### Placement
Start a placement chat and Leo asks one adaptive question at a time, going harder or easier based on your answers. After a few answered questions it records your **level** (Beginner → Expert), a **starting point**, a first set of knowledge-map concepts, and "learn next" points. The app refuses to set a level until you've answered at least three questions.

### Chat with Leo
There are no modes to pick. Ask a question, say "teach me…", or say "quiz me" and Leo adapts. Features include:

- Streaming replies with Markdown, GFM tables and LaTeX math (KaTeX)
- Tappable multiple-choice **ask** cards and graded **quiz** cards, with the option to dispute a wrong answer key (the stat change is reverted)
- Multiple threads per project, each with its own model choice
- Retry on errors and stop-while-streaming
- Context chips so you can see (and toggle off) what is being shared with Leo
- Voice dictation where the browser supports the Web Speech API
- `@` mentions for exact PDF pages, images, audio segments (15 / 30 / 60 s presets) and other library files

### Library
Upload or drag in:

| Kind | Formats | Notes |
| --- | --- | --- |
| PDF | `.pdf` | Page thumbnails, text indexed on-device for grounding |
| Image | PNG, JPEG, GIF, WebP, AVIF, BMP, SVG | Can be shared with Leo for vision-capable models |
| Audio | MP3, M4A, WAV, OGG, FLAC, Opus… | Waveform preview and player; built-in recorder |
| Video | MP4, MOV, WebM, M4V, MKV… | Poster-frame thumbnails |
| Text | `.txt`, `.md` | Indexed for grounding |

Single files up to 512 MB are accepted. The PDF reader renders pages through a priority queue that releases off-screen pages, which keeps memory use safe on phones (including iOS Safari's canvas limits).

### Notes
A live-preview Markdown editor: each block renders as formatted text and becomes editable source when tapped. Supports GFM, task-list checkboxes you can tick in place, and `$math$` / `$$math$$`. Lists continue automatically. Leo can create notes or edit an existing one (only when it has the note's id).

### Sketches
Full [Excalidraw](https://excalidraw.com) canvases, saved locally with thumbnails. Leo can generate a sketch from a simple JSON description of shapes, arrows and labels.

### Knowledge map, points and review
- **Knowledge map:** a concept tree with four statuses: *Not started*, *Learning*, *Shaky*, *Solid*.
- **Points card:** a shared memory between you and Leo, with three kinds: *Be careful about*, *Not understood yet*, *Learn next*.
- **Progress sheet:** level, recent accuracy, mastery, concepts due for review, and a one-tap **Review** or **Quiz me**.
- **Spaced review:** each correct answer pushes the next review further out (the interval grows by about 2.2× up to 60 days); a wrong answer brings the concept back within about half a day. Two consecutive correct answers promote a *learning* concept to *solid*; a miss on a *solid* concept demotes it to *shaky*.

### Memories
Leo can save durable facts you clearly state or confirm (level, goals, preferences, strengths, weak spots, facts), either globally or per project. Open **Memories** to view, add, edit or delete them. Leo is instructed never to store passwords, keys, medical details, precise location or other sensitive information.

### Search
A global search sheet looks across notes, sketches, library files, indexed PDF text, chats, points and concepts, with highlighted snippets.

### Learning browser
A lightweight in-app reader for reference pages:

- Pages that allow cross-origin reads (Wikipedia, MDN and most docs sites) are fetched, parsed off-document with `DOMParser` (no scripts run), and shown as clean readable text with links.
- Pages that block this can be captured with the **Leo Page Companion** bookmarklet, which you run yourself on the page.
- **Add-ons** are user-run bookmarklets. Two are built in (*YouTube Focus* and *Quiet article*) and you can add your own. Harness never injects code into another site on its own.
- Leo only sees a page's text after you capture or share it.

### Settings
Theme (light / dark / system), reduce motion, Leo's look, enter-to-send, share-media-by-default, confirm-before-delete, provider management, backup/restore, and Developer mode.

---

## Leo's action protocol

Leo's tools are plain JSON inside fenced `action` blocks. The app only executes payloads that pass validation; payloads are data only (never code).

| Action | Purpose |
| --- | --- |
| `ask` | Tappable multiple-choice question with no right answer |
| `quiz` | Graded question with answer index, explanation and the concept it tests |
| `assess` | Grade a free-text answer so it counts toward the map and review schedule |
| `set_level` | Record placement result (gated on 3+ answered questions) |
| `map` | Add or update a concept and its status / parent |
| `point` / `resolve_point` | Add or resolve an item on the points card |
| `note` | Create a note, or edit one by id |
| `sketch` | Create an Excalidraw sketch (rectangles, ellipses, diamonds, arrows, text; max 40 elements) |
| `browser` | Open a public HTTPS reference page in the learning browser |
| `memory` | Save, update or delete a long-term learner memory |
| `set_avatar` | Change Leo's shape, color or expression |

Rules enforced by the prompt and parser include at most one ask/quiz per message (placed at the end), valid JSON only, and no revealing quiz answers before the learner responds.

---

## Privacy and data

- **No server, no telemetry, no accounts.** Data never leaves your device except for requests to the AI provider you set up and pages you choose to open in the learning browser.
- The app ships with a strict `Content-Security-Policy`: scripts and styles are same-origin or inline, `object-src 'none'`, `base-uri 'none'`, `form-action 'none'`. `connect-src` allows `http:` / `https:` so you can reach any provider, including local ones.
- **Selection capture is conservative.** Text you highlight is shared with Leo only inside regions explicitly marked as shareable (notes, PDF text, Markdown). It is never captured from password or sensitive inputs, Settings, provider editors, or the chat panel, and it is cleared when you leave the screen.
- The app requests **persistent storage** so the browser is less likely to evict your data. Settings shows whether it was granted.

---

## Backups

**Settings → Data → Export backup** writes a JSON file containing all tables, every library file (as base64), and relevant local settings. You can choose whether to include API keys. Without keys, existing keys on the importing device are preserved.

**Import** merges a backup back in. Backups made by a newer schema version are rejected, and files over ~1.5 GB can't be imported in a browser tab. Export streams from many Blob parts so large libraries don't need one giant in-memory string.

If the browser can't guarantee persistence, **export backups regularly**.

---

## Tech stack

| Area | Choice |
| --- | --- |
| UI | React 19, TypeScript 5.9 |
| Build | Vite 7, `vite-plugin-singlefile` |
| Styling | Tailwind CSS 4 |
| State | Zustand 5 |
| Storage | Dexie 4 (IndexedDB) + `dexie-react-hooks` |
| Markdown / math | `react-markdown`, `remark-gfm`, `remark-math`, `rehype-katex`, KaTeX |
| PDF | `pdfjs-dist` |
| Sketching | `@excalidraw/excalidraw` |
| Icons / fonts | `lucide-react`, Inter Variable (bundled locally) |
| Tests | Vitest, `fake-indexeddb`, jsdom |

---

## Project structure

```
.
├── index.html                 # App shell, CSP, PWA manifest, early theme application
├── vite.config.ts             # React + Tailwind + single-file output, "@" → src alias
├── vitest.config.ts
├── stubs/
│   └── mermaid-to-excalidraw/ # Local stub that disables Mermaid import to keep the bundle small
├── qa/                        # Playwright-based visual and PDF-reader QA scripts
└── src/
    ├── App.tsx                # Router, selection capture, background indexing
    ├── main.tsx
    ├── screens/               # Home, Project, Settings, Memories, DevAvatars
    │   └── project/           # Library, Notes, Sketches
    ├── components/            # Chat panel, PDF viewer, recorder, knowledge map, progress,
    │                          # search, learning browser, avatar, UI primitives
    └── lib/
        ├── db.ts              # Dexie schema (v3), types, migrations, helpers
        ├── ai/
        │   ├── prompt.ts      # Leo's system prompt and tool documentation
        │   ├── actions.ts     # Action parsing, validation, execution, spaced review
        │   ├── providers.ts   # OpenAI-compatible / Anthropic / Gemini streaming clients
        │   └── mock.ts        # Developer mock provider
        ├── chat.ts            # Threads, sending, streaming, answering quizzes
        ├── retrieval.ts       # Chunking, BM25 ranking, background PDF indexing
        ├── media.ts           # File ingest, PDF/audio/video helpers
        ├── reader.ts          # Cross-origin page reader for the learning browser
        ├── backup.ts          # Export / import
        ├── avatar.ts          # Procedural avatar model and states
        └── …                  # router, settings, store, blocks, mentions, speech, etc.
```

The schema uses versioned Dexie migrations. **Never edit an existing version block.** Add a new `this.version(n)` with only the changed stores and an `.upgrade()` for data.

---

## Development

```bash
npm run dev       # start the dev server
npm run build     # production single-file build into dist/
npm run preview   # preview the production build
npm test          # run the Vitest suite
```

Unit tests cover the action parser and executor, spaced-review logic, Markdown block editing, `@`-mentions, the page reader, model labels, theme contrast, and core database behavior.

### Notes for contributors

- `@excalidraw/mermaid-to-excalidraw` is replaced by a local stub (`file:stubs/mermaid-to-excalidraw`) to keep the single-file bundle small. Mermaid conversion is intentionally unavailable.
- `window.EXCALIDRAW_ASSET_PATH` is set in `index.html` so Excalidraw doesn't fetch fonts from a CDN.
- Developer mode exposes the mock provider, forced avatar states, simulated errors and latency, and an avatar board at `#/dev/avatars`.
- Routes are hash-based (`#/p/:projectId/library`, `#/settings`, `#/memories`, …), so the app works on any static host without rewrite rules.

---

## Visual QA tooling

The `qa/` folder contains Playwright-based scripts (see [`qa/README.md`](qa/README.md)):

- `shoot.mjs` seeds a realistic project (notes, an AI note, concepts, points, and generated PNG / PDF / WAV / WebM files uploaded through the real UI) and screenshots every key screen in light and dark at phone size.
- `probe.mjs` and `webkit-pdf.mjs` stress-test the PDF reader with a 12-page mixed text/scan document in Chromium and WebKit.

```bash
npm run build
npx vite preview --port 4173 &
node qa/shoot.mjs before
```

These need a Chromium binary and `playwright-core`. Configure them with the `CHROME`, `PW_CORE` and `QA_BASE` environment variables. Output goes to `qa/out/` (git-ignored).

---

## Known limitations

- **Browser storage is the database.** Clearing site data, or aggressive eviction without persistent storage, deletes your work. Export backups.
- **Browser-only networking.** Providers must allow CORS (or be proxied). Some do not.
- **Large imports** need the whole backup in memory (limit about 1.5 GB).
- **Scanned PDFs** without a text layer can be viewed but not indexed, so Leo can't read them unless you quote or attach the content.
- **Mermaid import** into sketches is disabled in this build.
- **The learning browser** is not a full browser engine. It reads pages that permit it and otherwise relies on the companion bookmarklet.
- **Dictation** depends on the browser's Web Speech API support.

---

## Contributing

Issues and pull requests are welcome. Before opening a PR:

1. Run `npm test` and `npm run build`.
2. Keep Dexie migrations additive (new version blocks only).
3. Keep Leo's action payloads data-only and validated.

---

## License

No license file is included yet. Add one (for example MIT) before publishing if you want others to be able to use and contribute to the project.
