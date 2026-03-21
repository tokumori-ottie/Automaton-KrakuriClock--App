# CLAUDE.md — Automaton-KrakuriClock--App

## Project Overview

**からくりとけい メーカー** (Karakuri Clock Maker) is a Japanese-language Progressive Web App (PWA) for children to create and play custom animated "karakuri" clocks. Users follow a 6-step wizard to customize clock appearance, choose animated characters, pick music, and then watch their clock animate with a mechanical-toy style sequence.

**App title:** `🕰️ からくりとけい メーカー`
**PWA name:** `からくりどけいメーカー`
**Language:** Japanese (ja)

---

## Related Documentation

- **[REQUIREMENTS.md](./REQUIREMENTS.md)** — 要求仕様書 (Japanese requirements specification v1.1): full product spec including wizard steps, animation types, music tracks, data schema, UI/UX guidelines, and future roadmap.

---

## Repository Structure

```
/
├── index.html        # Entire application: HTML + embedded CSS + embedded JS (~4,350 lines)
├── sw.js             # Service Worker (cache-first strategy, cache name: karakuri-v1)
├── manifest.json     # PWA manifest (portrait, standalone display)
├── icon-192.png      # PWA icon (192×192)
├── icon-512.png      # PWA icon (512×512)
├── README.md         # Project overview and quick start
└── REQUIREMENTS.md   # 要求仕様書 (full product requirements, Japanese)
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
| `#pgWiz` | Wizard | 6-step clock customization wizard |
| `#pgList` | List | Saved clocks list |
| `#pgPlay` | Play | Clock playback with animated sequence |
| `#pgDone` | Done | Celebration screen after playback |
| `#craftPage` | Craft overlay | Interactive making animation shown between wizard steps (overlay, not a SPA page) |

Page transitions use `transform: translateX()` with classes `.active`, `.left`, `.right` and `transition: 0.38s cubic-bezier(.4,0,.2,1)`.

The `goTo(id, direction)` function handles all page navigation.

### Wizard Steps (1–6)

The `draft` object accumulates wizard selections:

| Step | ID | Content |
|---|---|---|
| 1 | `#s1` | Clock name (text input, max 12 chars; presets + free input) |
| 2 | `#s2` | Clock body shape (round / panel / square / arch) |
| 3 | `#s3` | Theme color — face color, frame color / camera roll photo |
| 4 | `#s4` | Hand color, number style, number color |
| 5 | `#s5` | Character (emoji or photo) + animation type(s) |
| 6 | `#s6` | Music track (10 options) |

`showWizStep(n, withAnim)` switches between steps and optionally triggers the craft animation overlay. `wizStep` tracks the current step (1–6).

### Key Global State Variables

```javascript
let draft = {
  name: '',
  clockBody: 'round',      // 'round' | 'panel' | 'square' | 'arch'
  faceColor: '#ffffff',
  handColor: '#ff6b6b',
  frameColor: '#ffd93d',
  numColor: '#444444',
  numberStyle: 'arabic',
  character: '🐻',
  anims: ['dance'],
  music: 'chime',
  faceImage: null,         // base64 DataURL (face photo, optional)
  charImage: null,         // base64 DataURL (character photo, optional)
};
let wizStep = 1;           // Current wizard step (1–6)
let playData = null;       // Clock data during playback
let playReal = true;       // Use real time in playback
let audioCtx = null;       // AudioContext lazy-initialized via getAC()
```

### Clock Data Shape (LocalStorage: `kk_clocks`)

```javascript
{
  id: Number,              // Date.now() unique ID
  name: String,            // Clock name (max 12 chars)
  clockBody: String,       // 'round' | 'panel' | 'square' | 'arch'
  faceColor: String,       // HEX color
  faceImage: String|null,  // base64 DataURL
  frameColor: String,      // HEX color
  handColor: String,       // HEX color
  numColor: String,        // HEX color
  numberStyle: String,     // 'arabic' | 'roman' | 'dots' | 'none'
  character: String,       // Emoji
  charImage: String|null,  // base64 DataURL
  anims: String[],         // Animation ID array (multi-select)
  music: String,           // Music track ID
}
```

### LocalStorage Keys

| Key | Contents |
|-----|----------|
| `kk_clocks` | Saved clocks array (max 5 items) |
| `kk_stats` | `{ totalCreated: number, unlocked: string[] }` — creation count and unlocked hidden characters |
| `kk_meta` | Metadata object (first launch date, etc.) |

---

## Key Constants

```javascript
CLOCK_BODIES  // 4 clock shapes: round, panel, square, arch
FACE_COLORS   // 8 face color options (HEX)
HAND_COLORS   // 8 hand color options (HEX)
FRAME_COLORS  // 8 frame color options (HEX)
NUM_COLORS    // 8 number color options (HEX)
NUM_STYLES    // 4 number styles: arabic, roman, dots, none
CHARS         // 12 default emoji characters
HIDDEN_CHARS  // 4 unlockable characters (butterfly/dragon/star/mage)
ANIMS         // 7 animation types (see below)
MUSICS        // 10 music tracks
TITLES        // 5 rank titles based on creation count
```

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

| Name | Trigger | Purpose |
|------|---------|---------|
| `countPop` | Countdown 3→2→1 | Number appears with scale overshoot |
| `aDance` | Character loop | Wiggle + rotate |
| `aBounce` | Character loop | Vertical bounce |
| `aSpin` | Character loop | 360° rotation |
| `aWave` | Character loop | Wave gesture |
| `selSpring` | Wizard card/char selection | Scale spring to 1.12 |
| `dotSpring` | Color dot selection | Scale spring to 1.22 |
| `cardPop` | List card selection | Scale pop then settle to 1.0 |
| `previewFlash` | Color / style selection | Preview canvas brightens + scale bounce (0.32s) |
| `previewPop` | Character selection | Preview canvas large spring bounce (0.38s) |
| `spkFly` | All selections | Sparkle emoji flies upward |
| `donePop` | Done screen | Celebration emoji pulses |

Classes `.preview-flash` / `.preview-pop` are applied to the preview `<canvas>` element and removed after the animation ends. `void el.offsetWidth` is used to force reflow for re-triggering on consecutive selections.

### Class Naming

- Page wrappers: `#pgTop`, `#pgWiz`, `#pgList`, `#pgPlay`, `#pgDone`
- Wizard elements: `.wiz-head`, `.wiz-prog-wrap`, `.wiz-bar-fill`, `.step-title`, `.step-hint`
- Buttons: `.top-btn`, `.back-btn`, `.char-btn`, `.next-btn`
- Cards: `.anim-card`, `.music-card`, `.body-card`
- Color dots: `.cdot`
- Number style cards: `.sel-card`

---

## JavaScript Conventions

### Naming

- **Variables:** camelCase (`faceColor`, `numberStyle`, `playData`)
- **Functions:** camelCase with verb prefix (`startWizard()`, `goTo()`, `refreshPreview()`, `showWizStep()`)
- **DOM IDs:** camelCase or abbreviated (`pgTop`, `s1`, `s2`)
- **Constants:** UPPER_SNAKE_CASE arrays/objects (`CLOCK_BODIES`, `CHARS`, `MUSICS`)

### Canvas Drawing

```javascript
// Convention: center coordinates and radius
const CX = canvas.width / 2;
const CY = canvas.height / 2;
const R = Math.min(CX, CY) * 0.9;
```

Clock face, hands, and numbers are drawn entirely with Canvas 2D calls.
`drawClock(canvas, data, useReal, tH, tM, tS)` dispatches to type-specific drawers (`drawClockRound`, `drawClockPanel`, `drawClockSquare`, `drawClockArch`).

### Audio

Web Audio API is used exclusively — no audio files. The `AudioContext` is lazy-initialized:
```javascript
function getAC() { /* returns or creates AudioContext */ }
```
Sounds are synthesized from oscillators (sine, square, sawtooth waveforms) with gain envelopes.

**Selection feedback sounds** (`playSelectSound(type)`):

| type | Sound | Used for |
|------|-------|----------|
| `'click'` | カチッ（square wave + bandpass filter, ~70ms） | Color dot selection |
| `'pop'` | ポンッ（sine wave pitch drop, ~180ms） | Character selection |
| `'gear'` | ガチャッ（2-click burst, ~130ms） | Number style selection |
| `'note'` | ポロン（E5→G5 two-note, ~240ms） | Music selection |

### Wizard Selection Feedback Pattern

When a user selects an option in the wizard, always call:
```javascript
refreshPreviewWithFeedback(soundType); // plays sound + triggers preview animation
```
This wraps `refreshPreview()` + `playSelectSound()` + `triggerPreviewAnim()`.

### Animation Playback

Uses `requestAnimationFrame` with normalized `ease` values (0→1) and easing functions.

| Animation | Function | Description |
|-----------|----------|-------------|
| split | `startSplitAnim()` / `drawSplitOverlay()` | 4 sectors rotating independently at different speeds |
| door | `startDoorAnim()` / `drawDoorOverlay()` | 4 fan-shaped doors fold open, character pops in |
| hanabira | `startHanabiraAnim()` / `drawHanabiraOverlay()` | 6 alternating flower petals open, character pops in |

All playback animations follow a 3-phase pattern: `opening → showing → closing`.

### LocalStorage Helpers

```javascript
function loadSaved()      // returns kk_clocks array
function saveSaved(arr)   // writes kk_clocks array
function loadStats()      // returns kk_stats object
function saveStats(s)     // writes kk_stats object
function loadMeta()       // returns kk_meta object
function saveMeta(m)      // writes kk_meta object
```

### Preview Functions

```javascript
function refreshPreview()                  // redraws preview canvas for current wizStep
function refreshPreviewWithFeedback(type)  // refreshPreview + playSelectSound + triggerPreviewAnim
function startPreviewTick()                // RAF loop for real-time clock hands (steps 3–4)
function stopPreviewTick()                 // cancels RAF loop
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
- Test all 6 wizard steps
- Verify selection sounds play on color / character / number style / music selection
- Verify preview canvas flash/pop animation on selection
- Test save/load from LocalStorage
- Test all 3 clock playback animations (split / door / hanabira)
- Test PWA install flow (manifest + service worker)
- Test image upload and crop modal (face photo + character photo)
- Test hidden character unlock flow (create 3 clocks → 🦋 appears)

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
9. **Preview animation re-trigger:** When adding `.preview-flash` / `.preview-pop` to the same element consecutively, always call `void el.offsetWidth` before adding the class to force a reflow and restart the animation.

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
