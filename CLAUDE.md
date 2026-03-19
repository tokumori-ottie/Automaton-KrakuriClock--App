# CLAUDE.md — Automaton-KrakuriClock--App

## Project Overview

**からくりとけい メーカー** (Karakuri Clock Maker) is a Japanese-language Progressive Web App (PWA) for children to create and play custom animated "karakuri" clocks. Users follow a 5-step wizard to customize clock appearance, choose animated characters, pick music, and then watch their clock animate with a mechanical-toy style sequence.

**App title:** `🕰️ からくりとけい メーカー`
**PWA name:** `からくりどけいメーカー`
**Language:** Japanese (ja)

---

## Repository Structure

```
/
├── index.html        # Entire application: HTML + embedded CSS + embedded JS (~3,260 lines)
├── sw.js             # Service Worker (network-first cache strategy)
├── manifest.json     # PWA manifest (portrait, standalone display)
├── icon-192.png      # PWA icon (192×192)
├── icon-512.png      # PWA icon (512×512)
└── README.md         # Minimal: "- からくり時計作成アプリ"
```

**No build system. No package.json. No dependencies.** This is a zero-dependency, static-file web application.

---

## Technology Stack

| Category | Technology |
|---|---|
| Markup | HTML5 |
| Styling | Vanilla CSS3 (custom properties, grid, flexbox, transforms, animations) |
| Scripting | Vanilla JavaScript ES6+ |
| Graphics | Canvas 2D API |
| Audio | Web Audio API (oscillators, gain nodes, filters — no audio files) |
| Storage | LocalStorage (JSON, max 5 saved clocks) |
| PWA | Service Worker + Web App Manifest |
| Fonts | Google Fonts: "Fredoka One" (display), "Nunito" (body) |
| Platform | Browser-only, mobile-first, portrait orientation |

---

## Architecture: Single-File SPA

Everything lives in `index.html`. There is no bundler, no module system, all code runs in the global scope.

### Pages (managed as absolutely-positioned divs)

| ID | Name | Purpose |
|---|---|---|
| `#pgTop` | Top/Home | Landing page with "新しく作る" and "つくったとけい" buttons |
| `#pgWiz` | Wizard | 5-step clock customization wizard |
| `#pgList` | List | Saved clocks list |
| `#pgPlay` | Play | Clock playback with animated sequence |
| `#pgDone` | Done | Celebration screen after playback |

Page transitions use `transform: translateX()` with classes `.active`, `.left`, `.right` and `transition: 0.38s cubic-bezier(.4,0,.2,1)`.

The `goTo(id, direction)` function handles all page navigation.

### Wizard Steps (1–5)

The `draft` object accumulates wizard selections:

| Step | Content |
|---|---|
| 1 | Clock name (text input) |
| 2 | Face color selection |
| 3 | Character (emoji) selection |
| 4 | Music track selection |
| 5 | Number style selection |

`buildWizUI(step)` dynamically generates each step's UI via template literals. `wizStep` tracks current step.

### Key Global State Variables

```javascript
let draft = {};        // Current clock being configured in the wizard
let wizStep = 1;       // Current wizard step (1-5)
let playData = {};     // Clock data during playback
// Audio context lazy-initialized via getAC()
```

### Clock Data Shape (LocalStorage)

```javascript
{
  name: String,          // Clock name
  faceColor: String,     // CSS color / gradient class
  char: String,          // Emoji character
  music: String,         // Music track ID
  numStyle: String,      // Number display style
  photo: String,         // Base64 image (optional)
  id: Number             // Timestamp ID
}
```

Saved clocks are stored in `localStorage` under a single key as a JSON array. Max 5 clocks enforced.

---

## CSS Conventions

### Color Variables (`:root`)

```css
--red: #ff6b6b
--yellow: #ffd93d
--green: #6bcb77
--blue: #4d96ff
--purple: #c77dff
--orange: #ff9a3c
--bg: #fff8f0
```

### Fluid Typography

Use `clamp()` for responsive font sizes:
```css
font-size: clamp(2rem, 9vw, 3rem);
```

### Keyframe Animations

Defined animations: `countPop`, `aDance`, `aBounce`, `aSpin`, `aWave` — used for character and UI feedback.

### Class Naming

- Page wrappers: `#pgTop`, `#pgWiz`, `#pgList`, `#pgPlay`, `#pgDone`
- Wizard elements: `.wiz-head`, `.wiz-prog-wrap`, `.wiz-bar-fill`, `.step-title`, `.step-hint`
- Buttons: `.top-btn`, `.back-btn`, `.char-btn`
- Cards: `.anim-card`, `.music-card`

---

## JavaScript Conventions

### Naming

- **Variables:** camelCase (`faceColor`, `numberStyle`, `playData`)
- **Functions:** camelCase with verb prefix (`startWizard()`, `goTo()`, `refreshPreview()`, `buildWizUI()`)
- **DOM IDs:** camelCase or abbreviated (`pgTop`, `s1`, `s2`)
- **Constants:** camelCase arrays/objects (`colorPalettes`, `charEmojis`, `musicTracks`)

### Canvas Drawing

```javascript
// Convention: center coordinates and radius
const CX = canvas.width / 2;
const CY = canvas.height / 2;
const R = Math.min(CX, CY) * 0.9;
```

Clock face, hands, and numbers are drawn entirely with Canvas 2D calls.

### Audio

Web Audio API is used exclusively — no audio files. The `AudioContext` is lazy-initialized:
```javascript
function getAC() { /* returns or creates AudioContext */ }
```
Sounds are synthesized from oscillators (sine, square, sawtooth waveforms) with gain envelopes.

### Animation

Uses `requestAnimationFrame` with normalized `ease` values (0→1) and easing functions. Two main animation styles:
- **Split animation:** 4-part rotating sectors
- **Door animation:** 4-petal opening sequence

### LocalStorage Helpers

```javascript
function loadSaved() { /* returns array from localStorage */ }
function saveSaved(arr) { /* writes array to localStorage */ }
```

---

## Service Worker

**File:** `sw.js`
**Cache name:** `karakuri-v1`
**Strategy:** Cache-first (serves from cache, falls back to network)
**Cached assets:** `/`, `/index.html`, `/manifest.json`

When updating the app, bump the cache version string `karakuri-v1` to invalidate old caches.

---

## Development Workflow

### Running Locally

Since there is no build step, serve the files with any static HTTP server:

```bash
# Python
python3 -m http.server 8080

# Node.js (if npx is available)
npx serve .

# VS Code: Live Server extension
```

Open `http://localhost:8080` in a mobile-emulation browser session (portrait, ~390px wide recommended).

### Making Changes

1. Edit `index.html` directly — CSS is in `<style>`, JS is in `<script>` at the bottom.
2. Refresh the browser (hard-refresh `Ctrl+Shift+R` to bypass Service Worker cache during dev).
3. Test on mobile viewport (portrait orientation, touch events).

### Testing

There is no automated test suite. Testing is manual and browser-based:
- Test all 5 wizard steps
- Test save/load from LocalStorage
- Test clock playback animation
- Test PWA install flow (manifest + service worker)
- Test image upload and crop modal

### No Linting / Formatting Tools

There is no ESLint, Prettier, or other tooling. Follow the existing code style when editing:
- 2-space indentation in CSS, compact single-line rules for short properties
- No semicolons are occasionally omitted in JS — match local style
- Template literals preferred over string concatenation

---

## PWA Notes

- **Orientation:** Portrait only (`"orientation": "portrait"` in manifest)
- **Display:** Standalone (no browser UI chrome)
- **Theme color:** `#ff6b6b` (red)
- **Background color:** `#fff8f0` (warm off-white)
- Icons are maskable (safe zone applies)

---

## Key Constraints and Gotchas

1. **Single HTML file:** All CSS and JS are embedded — do not create separate `.css` or `.js` files unless refactoring the entire project architecture.
2. **No npm:** Do not add package.json or npm dependencies.
3. **Mobile-first:** The app targets portrait mobile. Max content width is `480px`, centered.
4. **Japanese UI:** All user-facing text is in Japanese (hiragana/katakana/kanji). Keep additions consistent.
5. **LocalStorage limit:** Max 5 saved clocks enforced in application logic. Base64 photo data can be large — be mindful of storage limits.
6. **Service Worker caching:** During development, use hard-refresh or disable SW in DevTools to see changes immediately.
7. **Web Audio API:** Requires user gesture to initialize AudioContext (browser autoplay policy). `getAC()` must be called from within a click/touch handler.
8. **No external JS libraries:** All functionality is implemented from scratch using native browser APIs.

---

## Git

- **Main branch:** `main` (production)
- **Development:** Feature branches with prefix `claude/`
- **Commit signing:** GPG/SSH signing is enabled in this repo's git config
- **Remote:** `http://local_proxy@127.0.0.1:43505/git/tokumori-ottie/Automaton-KrakuriClock--App`

Push with:
```bash
git push -u origin <branch-name>
```
