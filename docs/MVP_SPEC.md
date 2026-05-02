---
dexid: 0xL1.01.00
dextype: spec
title: Lyrics Integrations — MVP Specification
version: 0.1.0
status: draft
author: k.greyZ
created: 2026-03-02
tags: [lyrics, web-component, offline-first, vanilla-js, fastapi, zenOS]
---

# Lyrics Integrations — MVP Specification

> Offline-first, device-sync-ready lyrics viewer, editor, and follow-along module.  
> Standalone web app. Embeddable Web Component. Zero framework lock-in.  
> APIs and ASR are optional enrichment layers — never dependencies.

---

## 0. Scope & Philosophy

- **Offline-first**: All core functionality works without internet, forever.
- **Local vault**: Your lyrics are yours. No forced cloud. Export anytime.
- **Device sync**: Optional FastAPI backend for multi-device personal sync.
- **Embeddable**: Ships as a `<lyrics-viewer>` Web Component, drop-in anywhere.
- **API-agnostic**: Genius, Musixmatch, etc. are optional plugins, not dependencies.
- **MIDI-ready**: Line change events exposed as `CustomEvent` for zenOS/midid integration.
- **ND-friendly**: Calm UI, no surprise motion, keyboard-first, WCAG 2.2 AA minimum.

---

## 1. Tech Stack

### Frontend

| Layer | Choice | Rationale |
|---|---|---|
| Language | Vanilla HTML5 / CSS3 / ESM | Zero runtime overhead, drop-in anywhere |
| Build | Vite (dev only) | HMR in dev; output is plain JS/CSS |
| Styling | CSS Custom Properties | Theme switching via `data-theme` attribute, no JS |
| State | Native `localStorage` + `IndexedDB` | No dep, persistent, offline |
| Web Component | Custom Elements v1 | `<lyrics-viewer>` embeddable in any stack |
| Icons | Inline SVG only | No external font/icon deps |

### Backend (Optional — Device Sync)

| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Python 3.11+ | Consistent with zenOS/dev-master stack |
| Framework | FastAPI | Already in zenOS ecosystem, async, OpenAPI auto-docs |
| DB | SQLite (local) → PostgreSQL (production) | Zero-config local; upgrade path to prod |
| Auth | JWT + optional passkey (WebAuthn) | zenOS WebAuthn standard |
| Sync | REST + optional WebSocket | Pull-based first; WS for live follow-along later |
| Export | JSON / CSV dump endpoints | Manual backup, no lock-in |

### Deployment

- **Offline-only**: Static `index.html`, no backend needed
- **Local**: `python -m uvicorn main:app` + open `index.html`
- **Self-hosted**: Docker Compose (app + db)
- **Hosted**: Vercel (frontend) + Railway/Fly.io (FastAPI)

---

## 2. Song Object Schema

This is the canonical Song entity. Every field is optional except `id` and `title`.

```typescript
interface Song {
  // ── Core Identity ──────────────────────────────────────────────
  id:                 string;       // UUID v4, generated on create
  title:              string;       // Required. Track title.
  artist:             string;       // Primary artist / act name
  artists:            string[];     // All credited artists (features etc.)
  album:              string;       // Album / release title
  album_artist:       string;       // Album-level artist (for compilations)
  track_number:       number;       // Position in album
  disc_number:        number;       // Multi-disc releases
  year:               number;       // Release year (YYYY)
  genre:              string[];     // Multiple genres allowed
  label:              string;       // Record label
  catalog_number:     string;       // Label catalog #
  isrc:               string;       // International Standard Recording Code
  upc:                string;       // Universal Product Code (album barcode)
  composer:           string;       // Songwriter / composer
  lyricist:           string;       // Lyric writer if different from composer
  remixer:            string;       // Remixer credit
  version:            string;       // e.g. "Radio Edit", "Extended", "VIP"
  language:           string;       // ISO 639-1, e.g. "en", "lv", "es"

  // ── Audio Properties ───────────────────────────────────────────
  bpm:                number;       // Beats per minute (float ok: 128.5)
  bpm_confidence:     number;       // 0.0–1.0, from analysis tool
  key:                string;       // Musical key, e.g. "Am", "F#m", "Cmaj"
  key_camelot:        string;       // Camelot wheel notation, e.g. "8A", "1B"
  energy:             number;       // 1–10 DJ energy rating
  danceability:       number;       // 0.0–1.0
  mood:               string;       // e.g. "dark", "euphoric", "melancholic"
  duration_ms:        number;       // Track duration in milliseconds
  bitrate:            number;       // kbps if known
  sample_rate:        number;       // Hz e.g. 44100, 48000

  // ── DJ / Performance Extensions ────────────────────────────────
  cue_points:         CuePoint[];   // See CuePoint interface below
  loops:              Loop[];       // See Loop interface below
  mix_in_ms:          number;       // Recommended mix-in timestamp (ms)
  mix_out_ms:         number;       // Recommended mix-out timestamp (ms)
  intro_end_ms:       number;       // Where the drop/vocals start
  outro_start_ms:     number;       // Where the outro begins
  key_compatible:     string[];     // Compatible Camelot keys for mixing
  bpm_range:          [number, number]; // Comfortable tempo-shift range

  // ── Lyrics ─────────────────────────────────────────────────────
  lyrics:             LyricsBlock[];  // Structured lyrics (see below)
  lyrics_raw:         string;       // Plain text fallback
  lyrics_source:      LyricsSource; // 'manual'|'asr'|'genius'|'musixmatch'|'import'
  lyrics_verified:    boolean;      // User has confirmed accuracy
  lyrics_language:    string;       // ISO 639-1, may differ from song language
  has_timestamps:     boolean;      // Whether line-level timestamps are available

  // ── Display / Playback Prefs ────────────────────────────────────
  scroll_speed:       number;       // Lines per minute in follow-along (default: bpm/8)
  scroll_offset_ms:   number;       // Manual timing offset correction in ms
  font_size_override: number;       // Per-song font size override (px)
  theme_override:     ThemeName;    // Per-song theme override

  // ── Organisation ───────────────────────────────────────────────
  tags:               string[];     // User-defined tags, freeform
  collections:        string[];     // Collection / playlist IDs
  rating:             number;       // 1–5 stars
  color:              string;       // Hex, for visual tagging in library
  comments:           string;       // Free text notes, cue notes, etc.
  explicit:           boolean;      // Explicit content flag
  is_favourite:       boolean;

  // ── Links ──────────────────────────────────────────────────────
  links: {
    source_url:       string;       // "Sauce" — where lyrics came from
    spotify_uri:      string;       // spotify:track:xxxx
    youtube_url:      string;
    soundcloud_url:   string;
    bandcamp_url:     string;
    apple_music_url:  string;
    beatport_url:     string;
    tidal_url:        string;
    discogs_url:      string;
    musicbrainz_id:   string;       // MusicBrainz recording MBID
    custom:           { label: string; url: string }[]; // Arbitrary extra links
  };

  // ── zenOS / DEX Integration ─────────────────────────────────────
  dex_path:           string;       // DEX address e.g. "0xL1.04.02"
  midid_cues:         MidiCue[];    // MIDI events tied to lyric lines
  zenOS_tags:         string[];     // zenOS engine tags: 'product'|'persona'|'investment'

  // ── File / Import Metadata ──────────────────────────────────────
  file_path:          string;       // Local file reference if applicable
  file_hash:          string;       // SHA256 of audio file for dedup
  artwork_url:        string;       // Album art URL or base64 data URI
  import_source:      string;       // e.g. 'rekordbox'|'serato'|'id3'|'manual'
  external_id:        Record<string, string>; // { genius: '123', musicbrainz: 'uuid' }

  // ── Audit ───────────────────────────────────────────────────────
  created_at:         string;       // ISO 8601
  updated_at:         string;       // ISO 8601
  last_played_at:     string;       // ISO 8601
  play_count:         number;
  revision:           number;       // Increments on every lyrics edit
}
```

### Supporting Interfaces

```typescript
interface LyricsBlock {
  id:      string;
  type:    'verse'|'chorus'|'bridge'|'hook'|'intro'|'outro'|
           'break'|'interlude'|'ad-lib'|'other';
  label:   string;       // Display label e.g. "Chorus 1", "Verse 2"
  lines:   LyricsLine[];
}

interface LyricsLine {
  id:             string;
  text:           string;
  timestamp_ms?:  number;    // Optional: start timestamp for follow-along
  speaker?:       string;    // For multi-artist tracks
  annotation?:    string;    // Inline note / translation
}

interface CuePoint {
  id:           string;
  label:        string;     // e.g. "Drop", "Build", "Hook"
  color:        string;     // Hex
  timestamp_ms: number;
  type:         'hot_cue'|'memory_cue'|'loop_in'|'loop_out';
}

interface Loop {
  id:       string;
  label:    string;
  start_ms: number;
  end_ms:   number;
  active:   boolean;
}

interface MidiCue {
  line_index:  number;     // Which lyrics line triggers this
  channel:     number;     // MIDI channel 1–16
  note:        number;     // MIDI note 0–127
  velocity:    number;     // 0–127
  dex_address: string;     // Optional DEX target
}

type LyricsSource = 'manual'|'asr'|'genius'|'musixmatch'|
                    'azlyrics'|'import'|'unknown';
type ThemeName    = 'dark'|'light'|'oled'|'glitchworks'|
                    'solarized'|'custom';
```

---

## 3. UI Modes

### 3.1 Read Mode
- Full-width, distraction-free lyrics display
- Section headers visually distinct (Verse / Chorus / Bridge etc.)
- Font size, line height, theme controls in collapsible toolbar
- Keyboard: `f` = follow-along, `e` = edit, `Esc` = close controls

### 3.2 Edit Mode
- Line-by-line `contenteditable` editor with section management
- Add / delete / reorder lines and blocks via drag or keyboard
- Auto-save to IndexedDB on blur (debounced 500ms)
- Version history: revision counter, one-click undo to previous save
- Full song metadata panel accessible without leaving the page

### 3.3 Follow-Along Mode
- Manual progress slider (0–100%) or keyboard advance
- Active window: current line (large) + 2 lines context above/below
- Scroll speed derived from `scroll_speed` field (default: `bpm / 8` lines/min)
- Keyboard: `Space` = next line, `←/→` = step, `r` = reset
- MIDI hook: `window.dispatchEvent(new CustomEvent('lyrics:line', { detail: { index, line } }))`
- Auto-scroll mode toggle: manual vs. timed

---

## 4. Themes

All themes implemented as CSS Custom Property sets on `<html data-theme="...">`. Zero JS for theme switching.

| Theme | Description |
|---|---|
| `dark` | Default. Near-black bg, light text, blue accent |
| `light` | White bg, dark text, accessible contrast |
| `oled` | Pure `#000000` bg, optimised for OLED screens |
| `glitchworks` | `#0a0a0a` bg, cyan `#22d3ee` + pink `#ec4899`, mono font, **no animations** |
| `solarized` | Classic Solarized Dark |
| `custom` | User-defined via CSS var overrides in settings |

Per-song `theme_override` takes precedence over global setting.

---

## 5. Data Layer

### 5.1 Local Storage (Offline-First)

- **IndexedDB** via native API — no wrapper libraries
- Object store: `songs` — keyed by `id` (UUID)
- Object store: `collections` — playlists / curated sets
- Object store: `settings` — UI prefs, theme, default scroll speed
- On first load: seed with a demo song so the UI is never empty

### 5.2 Import / Export (Manual Dumps)

```
Export formats:
  JSON  → Full song array, all fields, human-readable
  CSV   → Flat metadata only (no nested lyrics), for spreadsheets
  LRC   → Standard .lrc lyric format with timestamps
  MD    → Markdown with YAML frontmatter + lyrics body

Import formats:
  JSON  → Full restore from export
  LRC   → Timestamped lyrics import
  TXT   → Plain text lyrics, no timestamps
  ID3   → Extract tags from audio file drag-drop (client-side)
```

### 5.3 FastAPI Sync Backend (Optional)

```
POST   /api/songs              Create song
GET    /api/songs              List (paginated, filterable by tag/artist/key/bpm)
GET    /api/songs/{id}         Get by ID
PUT    /api/songs/{id}         Update
DELETE /api/songs/{id}         Soft delete
GET    /api/songs/{id}/history Revision history

POST   /api/export             Full JSON dump
GET    /api/export/csv         Flat CSV metadata
POST   /api/import             Bulk import from JSON dump

GET    /api/sync/status        Last sync timestamp per device
POST   /api/sync/push          Push local changes to server
GET    /api/sync/pull          Pull remote changes since timestamp

POST   /api/auth/login         JWT login
POST   /api/auth/refresh       Refresh JWT
```

Sync strategy: **last-write-wins per song** (by `updated_at`).  
Conflict flag surfaced to UI when both sides changed since last sync.

---

## 6. Web Component API

```html
<!-- Standalone viewer, read-only -->
<lyrics-viewer
  song-id="uuid-here"
  mode="read"
  theme="glitchworks"
  font-size="18"
></lyrics-viewer>

<!-- Follow-along with MIDI output -->
<lyrics-viewer
  song-id="uuid-here"
  mode="follow"
  midi-out="true"
></lyrics-viewer>
```

```js
// JS API
const viewer = document.querySelector('lyrics-viewer');
viewer.loadSong(songObject);  // Load from JS object directly
viewer.setMode('edit');       // Switch mode programmatically
viewer.setProgress(0.5);      // Seek to 50% in follow-along

// Events emitted
viewer.addEventListener('lyrics:line',   e => {});
viewer.addEventListener('lyrics:save',   e => {});
viewer.addEventListener('lyrics:export', e => {});
```

---

## 7. File Structure

```
lyrics-integrations/
├── index.html                    ← Standalone demo / app shell
├── vite.config.js
├── package.json
│
├── src/
│   ├── main.js                   ← App entry, registers Web Component
│   ├── web-component.js          ← <lyrics-viewer> Custom Element
│   │
│   ├── components/
│   │   ├── LyricsViewer.js       ← Read mode
│   │   ├── LyricsEditor.js       ← Edit mode
│   │   ├── FollowAlong.js        ← Follow-along mode
│   │   ├── SongMeta.js           ← Metadata editor panel
│   │   ├── Toolbar.js            ← Theme / font / mode controls
│   │   └── ImportExport.js       ← Import/export UI
│   │
│   ├── stores/
│   │   ├── db.js                 ← IndexedDB adapter
│   │   ├── songs.js              ← Song CRUD + local state
│   │   └── sync.js               ← FastAPI sync adapter (optional)
│   │
│   ├── themes/
│   │   ├── base.css
│   │   ├── dark.css
│   │   ├── light.css
│   │   ├── oled.css
│   │   ├── glitchworks.css       ← cyan/pink, mono, static, no animations
│   │   └── solarized.css
│   │
│   └── utils/
│       ├── schema.js             ← Song schema + default factory fn
│       ├── id3.js                ← Client-side ID3 tag reader
│       ├── lrc.js                ← LRC parse/serialize
│       ├── export.js             ← JSON/CSV/MD export
│       └── midi.js               ← MIDI cue event emitter
│
├── api/                          ← FastAPI backend (optional)
│   ├── main.py
│   ├── models.py
│   ├── routes/
│   │   ├── songs.py
│   │   ├── sync.py
│   │   └── auth.py
│   └── db/
│       └── schema.sql
│
├── docs/
│   └── MVP_SPEC.md               ← This file
│
├── tests/
│   ├── schema.test.js
│   └── lrc.test.js
│
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── ci.yml
```

---

## 8. DJ Software Interop

| Source | Format | Fields Mapped |
|---|---|---|
| Rekordbox | XML export | BPM, key, cue points, energy, comments, file path |
| Serato | `_Serato_` ID3 frames | Cue points, loops, BPM, key |
| Traktor | NML export | BPM, key, cue points, rating |
| Beatport | CSV purchase history | Title, artist, BPM, key, genre, beatport_url |
| MusicBrainz | Picard tags | MBID, ISRC, label, catalog, year |
| plain ID3v2 | Audio file drag-drop | All standard + TXXX custom frames |

---

## 9. Phased Delivery

### Phase 1 — Viewer Core (MVP)
- [ ] Vite scaffold + Web Component shell
- [ ] IndexedDB store with full Song schema
- [ ] Read mode with GlitchWorks + 3 other themes
- [ ] Edit mode with auto-save and revision counter
- [ ] Follow-along mode with keyboard controls + MIDI event hook
- [ ] JSON import / export
- [ ] Demo song pre-loaded on first launch

### Phase 2 — Metadata & DJ Features
- [ ] Full metadata editor (all song fields)
- [ ] ID3 tag reader (audio file drag-drop)
- [ ] LRC import/export
- [ ] Rekordbox XML import
- [ ] CSV export
- [ ] Collections / playlists

### Phase 3 — Backend Sync
- [ ] FastAPI backend scaffold
- [ ] JWT auth
- [ ] REST CRUD + bulk import/export endpoints
- [ ] Pull-based sync (last-write-wins)

### Phase 4 — Enrichment
- [ ] Genius / Musixmatch adapter (optional plugin)
- [ ] Local ASR transcription pipe (Whisper)
- [ ] MusicBrainz metadata lookup
- [ ] Conflict UI for sync disagreements

---

## 10. Phase 1 Acceptance Criteria (DoD)

- [ ] App works fully offline — zero network requests on load
- [ ] `<lyrics-viewer>` embeds in plain HTML with one `<script>` tag
- [ ] All 3 modes (read / edit / follow-along) functional
- [ ] GlitchWorks theme renders correctly, zero animations
- [ ] Song survives page refresh (IndexedDB persists)
- [ ] JSON round-trip: export → import restores all fields
- [ ] WCAG 2.2 AA: keyboard nav, visible focus, no unlabelled controls
- [ ] `prefers-reduced-motion` respected
- [ ] `lyrics:line` CustomEvent fires on line advance in follow-along
- [ ] CI passes on push

---

*DEX Path: `0xL1.01.00` — lyrics-integrations spec root*  
*v0.1.0 · 2026-03-02 · k.greyZ*
