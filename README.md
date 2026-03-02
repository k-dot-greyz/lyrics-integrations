# lyrics-integrations

> Offline-first lyrics viewer, editor, and follow-along module — for the zenOS ecosystem and the world.

## Features

- `<lyrics-viewer>` Web Component — drop into any page with one `<script>` tag
- 3 modes: **Read**, **Edit**, **Follow-Along**
- 5 themes including **GlitchWorks** (cyan/pink, static, no animations)
- Full offline support via IndexedDB — zero network dependency
- Rich song schema: ID3 tags, DJ extensions (BPM, key, Camelot, cue points, loops), MIDI cues, DEX path
- Import/Export: JSON, CSV, LRC, Markdown, plain ID3 drag-drop
- Optional FastAPI sync backend for multi-device use
- MIDI hook: `lyrics:line` CustomEvent on every line advance

## Quick Start

```bash
npm install
npm run dev
```

Open `http://localhost:5173`

## Embed Anywhere

```html
<script type="module" src="/dist/lyrics-viewer.js"></script>
<lyrics-viewer song-id="your-uuid" mode="read" theme="glitchworks"></lyrics-viewer>
```

## Docs

- [MVP Spec](docs/MVP_SPEC.md) — Full spec, song schema, stack, phases, acceptance criteria

## Stack

- Vanilla HTML5 / CSS3 / ESM — zero runtime framework
- Vite (dev build only)
- IndexedDB (offline storage)
- FastAPI (optional sync backend)
- Custom Elements v1 Web Component

## DEX Path

`0xL1.01.00`

---

*Part of the zenOS ecosystem · [dev-master](https://github.com/k-dot-greyz/dev-master)*
