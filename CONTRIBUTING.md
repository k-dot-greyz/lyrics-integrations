# Contributing to lyrics-integrations

Welcome! This repository is an offline-first lyrics viewer, editor, and follow-along module for the zenOS ecosystem and standalone embeds. Thanks for keeping the architecture clean and the submodule boundary intact.

For product scope, song schema, and phased delivery, see [README.md](./README.md) and [docs/MVP_SPEC.md](./docs/MVP_SPEC.md).

---

## Repository overview

**lyrics-integrations** ships as a vanilla ESM Web Component (`<lyrics-viewer>`), a Vite-powered dev shell, IndexedDB-backed local vault, and an optional FastAPI sync backend. External lyric APIs (Genius, Musixmatch, etc.) are optional plugins—never hard dependencies.

| Area | Technology |
|------|------------|
| Frontend | Vanilla HTML5 / CSS3 / ESM (Custom Elements v1) |
| Build | [Vite](https://vitejs.dev/) (dev/HMR; distributable is plain JS/CSS) |
| Styling | CSS custom properties + `data-theme` (no runtime CSS-in-JS framework) |
| Local state | `localStorage` + [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) |
| Unit tests | [Vitest](https://vitest.dev/) + happy-dom + fake-indexeddb (when `tests/` is present) |
| Optional backend | Python 3.11+ · [FastAPI](https://fastapi.tiangolo.com/) · SQLite → PostgreSQL upgrade path |
| Standards | [WCAG 2.2 AA](https://www.w3.org/TR/WCAG22/), [LRC](https://en.wikipedia.org/wiki/LRC_(file_format)), `lyrics:line` `CustomEvent` for MIDI/karaoke hooks |

**DEX path:** `0xL1.01.00` (spec root in [docs/MVP_SPEC.md](./docs/MVP_SPEC.md)).

### Layout

**Current (`main` — spec scaffold):**

```
lyrics-integrations/
├── README.md
├── CONTRIBUTING.md          ← this file
└── docs/
    └── MVP_SPEC.md        ← canonical schema, phases, acceptance criteria
```

**Target (Phase 1+ — see MVP spec §7):**

```
lyrics-integrations/
├── index.html
├── vite.config.js
├── package.json
├── src/
│   ├── main.js / main.ts
│   ├── web-component.js     # <lyrics-viewer>
│   ├── components/        # Read, Edit, Follow-along, toolbar, import/export
│   ├── stores/            # IndexedDB, songs CRUD, optional sync adapter
│   ├── themes/            # dark, oled, glitchworks, solarized, light
│   └── utils/             # schema, lrc, id3, export, midi events
├── api/                   # optional FastAPI (Phase 3)
├── tests/                 # vitest (schema, lrc, player, etc.)
├── docs/
└── .github/workflows/ci.yml
```

As implementation lands, prefer the **pure-module split** used in active branches: DOM wiring in a single entry (`main.ts`), everything else unit-testable without a browser (see project hub notes in the parent `dev-master` repo).

---

## 1. The Prime Directive: Pure Code in Submodules, Guides in Superproject

`lyrics-integrations` is vendored as a git submodule (for example under `dev-master` at `dex/09-repos/lyrics-integrations`). We enforce a strict boundary between the standalone upstream repo and any parent monorepo.

### The boundary violation rule

**NEVER commit internal parent-workspace documentation, fork-specific setup notes, or monorepo SOPs into this repository.**

* **Why?** This repo must remain portable, upstreamable, and free of private orchestration noise.
* **The standard:** Keep this repository limited to **application code, tests, product-facing docs, and the MVP spec**. Allowed at repo root: `README.md`, `CONTRIBUTING.md`, and `docs/MVP_SPEC.md` (product contract—not monorepo workflow).
* **Move to superproject:** Private guides, `dev-master` git/submodule workflows, agent handoffs, and switchboard routing docs belong under `dev-master/dex/03-docs/guides/` (see [SUBMODULE_CONTRIBUTING_WORKFLOW.md](https://github.com/k-dot-greyz/dev-master/blob/main/dex/03-docs/guides/SUBMODULE_CONTRIBUTING_WORKFLOW.md)).

---

## 2. GlitchWorks Agnostic Architecture Protocol (`/architecture-base`)

All features must follow the **GlitchWorks Agnostic Architecture Protocol** so the viewer stays embeddable, offline-capable, and swappable across hosts (plain HTML, zenOS shells, DJ tools).

### 2.1. Zero hardcoding (dynamic state configuration)

* **Rule:** No magic ports, fixed API hosts, or hard-coded storage keys in domain logic.
* **Application:** Resolve Vite dev port, FastAPI base URL, IndexedDB database name, and theme defaults via injected config, environment variables, or constructor options—not scattered literals in `src/`.

### 2.2. Polymorphism by default (interface-driven contracts)

* **Rule:** Depend on abstractions, not concretions.
* **Application:** Lyric sources (manual vault, LRC import, optional Genius/Musixmatch adapters), storage (`IndexedDB` vs in-memory test double), and playback clocks (`AudioContext`, external MIDI, manual keyboard advance) must implement narrow TypeScript interfaces. UI and karaoke engine code talk only to those interfaces.

### 2.3. Open piping (strict inter-process communication)

* **Rule:** Communicate via typed events and payloads, not shared mutable singletons.
* **Application:** Line advances emit `lyrics:line` `CustomEvent` with a stable detail schema. Optional sync with FastAPI uses REST/WebSocket contracts defined in `api/`—hosts must not reach into component internals.

### 2.4. Boundary validation (the hostile edge)

* **Rule:** Never trust imports, drag-drop files, or API responses.
* **Application:** Validate Song objects, LRC parses, and ID3-derived metadata at the edge (import UI, API routes) before persisting to IndexedDB. Reject malformed payloads with typed errors—no silent partial writes.

### 2.5. State hydration and dehydration

* **Rule:** Export and restore truth as snapshots.
* **Application:** JSON/CSV/LRC export must round-trip per MVP spec §10. Session/follow-along position should be serializable so refresh and multi-device sync (Phase 3) can resume without drift.

### 2.6. Graceful degradation (predictable failure)

* **Rule:** Offline-first means network absence is normal, not exceptional.
* **Application:** If sync backend or enrichment APIs are unreachable, keep read/edit/follow-along functional on local vault data; surface non-blocking status, never blank the UI with an uncaught exception.

### 2.7. Agnostic telemetry and observability

* **Rule:** Core logic must not assume a telemetry vendor.
* **Application:** Use an injectable logger or `console` behind an interface. No forced analytics, login, or paywall telemetry—aligned with MVP philosophy (see README).

---

## 3. Coding conventions

* **Vanilla ESM first:** No React/Vue/Svelte in the viewer core; Custom Elements v1 for embeddability.
* **TypeScript where tests exist:** Prefer `.ts` in `src/` and `tests/` with strict checking once `package.json` is present.
* **Accessibility:** Keyboard-first flows, semantic landmarks, ARIA toolbar pattern, `aria-live="polite"` for line changes, visible focus rings, `prefers-reduced-motion` (GlitchWorks theme stays static—no surprise motion).
* **Themes:** CSS variables + `data-theme`; per-song overrides via schema fields in MVP spec §2.
* **Offline vault:** IndexedDB is source of truth; network is optional enrichment.
* **Conventional commits:** `type(scope): message` (for example `feat(lrc): parse multi-stamp lines with offset`).

---

## 4. Local quality gates

### Spec-only checkout (no `package.json` yet)

Before opening a docs-only PR:

```bash
git status
git diff
```

Ensure you are not adding monorepo-only markdown or parent-workspace paths.

### Application scaffold present

When `package.json` and `tests/` exist on your branch:

```bash
npm install
npm run check        # tsc / typecheck (when scripted)
npm test             # vitest run
npm run dev          # Vite dev server — default http://localhost:5173
```

Optional manual smoke (Phase 1 DoD — see MVP spec §10):

- App loads with **zero network** requests
- Demo song persists across refresh
- Read / Edit / Follow-along modes and GlitchWorks theme
- JSON export → import round-trip
- `lyrics:line` fires on line advance in follow-along

### Optional FastAPI backend (Phase 3+)

```bash
cd api
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

Keep API URL configurable; never hard-code in frontend domain modules.

---

## 5. Fork-and-PR workflow

### Step 1: Configure remotes

```bash
git remote -v
# If upstream is missing:
git remote add upstream https://github.com/k-dot-greyz/lyrics-integrations.git
```

### Step 2: Branch from latest default branch

```bash
git fetch upstream
git checkout -b feat/your-feature-name upstream/main
```

Use prefixes: `feat/`, `fix/`, `docs/`, `refactor/`, `test/`.

### Step 3: Implement pure code changes

* No `.env` secrets, build artifacts, or `node_modules/` in commits.
* No monorepo workflow docs in this repo.
* Match [MVP_SPEC.md](./docs/MVP_SPEC.md) schema and phase scope.

### Step 4: Pre-commit submodule audit

1. **`git status`** — any markdown describing `dev-master` setup, submodule bump scripts, or agent SOPs? Move to superproject guides.
2. **`git diff --name-status upstream/main`** — only files relevant to your change?
3. **`git diff`** — remove debug logging, formatting-only churn, and commented-out experiments.

### Step 5: Commit and push

```bash
git commit -m "feat(viewer): short conventional summary"
git push -u origin HEAD
```

### Step 6: Open a pull request

```bash
gh pr create --repo k-dot-greyz/lyrics-integrations --base main \
  --title "feat(viewer): short summary" \
  --body "$(cat <<'EOF'
## Summary
- …

## Test plan
- [ ] `npm run check` (when present)
- [ ] `npm test`
- [ ] Manual offline smoke per MVP spec §10
EOF
)"
```

If `origin` already points at `k-dot-greyz/lyrics-integrations`, push to `origin` and open the PR the same way.

---

## 6. Cleaning up after a boundary leak

If internal monorepo docs were committed on your branch:

```bash
git reset --soft upstream/main
# Move misplaced docs to dev-master/dex/03-docs/guides/
git restore <unwanted-file>
git commit -m "feat(module): clean feature only"
git push origin your-branch --force
```

Only force-push branches you own and that are not merged upstream.

---

*Keep the vault local, the pipes typed, and the karaoke line dead-center. Thanks for contributing.*
