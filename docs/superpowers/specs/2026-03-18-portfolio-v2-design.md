# Portfolio v2 — Design Specification

**Author:** Matvei Tkhai + Claude
**Date:** 2026-03-18
**Status:** Draft
**Repo:** https://github.com/potamotus/matvei-site-2

---

## 1. Overview

A single-page interactive portfolio site built as a **scroll-driven flight through space**. The visitor travels through four spiral galaxies — each representing a life domain (Education, Research, Projects, Contact). Stars within galaxies are clickable and reveal content in sci-fi HUD frames with CRT effects.

**Tech stack:** Single HTML file, Three.js r160+ (self-hosted, WebGL2), vanilla JS, no build tools.

**Target:** Modern browsers (Chrome, Safari, Firefox), desktop + mobile.

---

## 2. Site Structure (Flight Path)

The visitor's journey follows a linear z-axis path through space:

```
Hero (z=0)
  ↓ scroll
Open space (acceleration zone)
  ↓
Galaxy 1: Education (purple, z ≈ -2000)
  ↓
Open space (acceleration zone)
  ↓
Galaxy 2: Research (blue, z ≈ -5000)
  ↓
Open space (acceleration zone)
  ↓
Galaxy 3: Projects (orange, z ≈ -8000)
  ↓
Open space (acceleration zone)
  ↓
Galaxy 4: Contact (green, z ≈ -11000)
  ↓
Hyperspace jump → back to Hero
```

Total scroll distance: ~1200vh mapped to camera z-position.

---

## 3. Hero Screen

### 3.1 Loading & Signal Intro

The signal intro doubles as a loading screen — Three.js initializes in the background while the intro plays.

**Sequence:**
1. Page loads → signal intro starts immediately (no Three.js dependency)
2. Three.js scene initializes in parallel (geometry, shaders, textures)
3. Intro finishes (~3s) → check if Three.js is ready
   - **Ready:** fade to hero
   - **Not ready:** hold on `LINK ESTABLISHED` text, add subtle `...` pulsing until ready
4. If WebGL unavailable entirely: show static fallback (see Section 10.4)

**Signal intro details:**
- Full-screen dark overlay with noise canvas
- 3 lines typed at 25ms per character:
  1. `SCANNING DEEP SPACE...`
  2. `SIGNAL ACQUIRED · 1420.405 MHz`
  3. `LINK ESTABLISHED`
- Noise intensity decreases with each line
- Click anywhere to skip (only if Three.js is ready)
- Duration: ~3 seconds total (minimum, extends if GPU is slow)

### 3.2 Hero Content
After intro fades (0.6s opacity transition), hero reveals:

| Element | Style | Delay |
|---------|-------|-------|
| Catalog label | IBM Plex Mono 10px, 0.07 opacity | 200ms |
| Name "MATVEI TKHAI" | Saira 600, clamp(38px, 6.5vw, 76px) | 350ms |
| Subtitle "economist · builder" | IBM Plex Mono 11px, 0.1 opacity | 950ms |
| Coordinates | IBM Plex Mono 9px, 0.04 opacity | 1100ms |
| Corner HUD (3 corners) | IBM Plex Mono 8px, 0.1 opacity | 500-750ms |
| Scroll hint | IBM Plex Mono 9px, 0.06 opacity | 1500ms |

### 3.3 Hero Visual Effects

**Background layers (bottom to top):**
1. `canvas#stars` — 300 twinkling stars (fixed, z-index: 0)
2. Binary stream columns — 12 vertical columns, 0.02 opacity (z-index: 1)
3. Hero noise canvas — 1/4 resolution grain, 0.03 opacity (z-index: 49)
4. Scanlines — 1px lines every 4px, 0.035 opacity, pulse animation (z-index: 50)
5. Horizontal grid — 0.5px lines every 40px, 0.035 opacity, flicker animation (z-index: 60)

**Name CRT shift effect:**
- Two sibling elements: original `.hero-name` + `.hero-name-shift`
- JS periodically (every 0.2–0.8s) activates 1–3 shift bands:
  - Band height: 3–8% of name height
  - Original text: band area cut via `clip-path: polygon()`
  - Shift copy: band shown via `clip-path: inset()` + `translateX(±3–8px)`
  - Duration: 60–140ms per band
- Whole-screen jitter: entire hero shifts ±1–3px every 1.5–4.5s for 40–100ms

**Chromatic aberration:** Breathing animation on name via `::before`/`::after` pseudo-elements with red/blue color shifts (subtle, ±0.4–1.5px drift on 3s cycle).

---

## 4. Scroll-Driven Flight

### 4.1 Camera System
- Three.js `PerspectiveCamera`, FOV 60, near 0.1, far 15000
- Camera moves along z-axis based on scroll position
- `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` — cap at 2x for performance
- Scroll position → camera z mapping is piecewise-linear with eased transitions between segments

### 4.2 Scroll Zone Map

| Scroll (vh) | Phase | Camera z | Speed | Notes |
|-------------|-------|----------|-------|-------|
| 0–50 | Hero | 0 | static | Hero visible, no flight |
| 50–120 | Transit 1 | 0 → -1500 | fast (21z/vh) | Star stretch active |
| 120–250 | Galaxy 1: Education | -1500 → -2500 | slow (7.7z/vh) | Stars clickable |
| 250–330 | Transit 2 | -2500 → -4500 | fast (25z/vh) | Star stretch |
| 330–460 | Galaxy 2: Research | -4500 → -5500 | slow (7.7z/vh) | Stars clickable |
| 460–540 | Transit 3 | -5500 → -7500 | fast (25z/vh) | Star stretch |
| 540–670 | Galaxy 3: Projects | -7500 → -8500 | slow (7.7z/vh) | Stars clickable |
| 670–750 | Transit 4 | -8500 → -10500 | fast (25z/vh) | Star stretch |
| 750–880 | Galaxy 4: Contact | -10500 → -11500 | slow (7.7z/vh) | Stars clickable |
| 880–950 | Exit | -11500 → -12500 | slow | Approach dark screen |
| 950–1000 | Hyperspace prompt | -12500 | static | "INITIATE RETURN JUMP" |

Transitions between fast/slow zones use `smoothstep` easing over ~10vh to avoid jarring speed changes.

Each galaxy has 3 phases:

```
[Approach] → [Inside] → [Exit]
  fast          slow        fast
```

**Approach phase:**
- Camera decelerates as galaxy grows ahead
- Galaxy visible as distant glow → spiral structure becomes clear
- Stars begin to resolve

**Inside phase:**
- Slow scroll, stars are large and clickable
- Galaxy structure surrounds the viewer
- HUD label shows galaxy name (e.g., "EDUCATION")

**Exit phase:**
- Camera accelerates away
- Star stretch effect begins (see 4.3)
- Galaxy shrinks behind

### 4.3 Star Stretch (Acceleration Effect)

Between galaxies, stars visually stretch along the z-axis:

**Implementation:**
- Custom vertex shader on star `Points` material
- When camera velocity exceeds threshold:
  - Each star's point is elongated into a line (using `gl_PointSize` stretch or geometry)
  - Stretch amount proportional to velocity
  - Stars near screen edges stretch more (radial effect)
- Duration: 1–2 seconds of scrolling at accelerated pace
- Easing: smooth ramp up, smooth ramp down

**Visual parameters:**
- Max stretch: 15–25px (screen space)
- Opacity during stretch: slightly brighter (1.2x)
- Background: subtle radial speed lines (optional, CSS overlay)
- **Note:** Verify `gl_ALIASED_POINT_SIZE_RANGE` at init. If max point size < 25, use instanced `THREE.LineGeometry` for stretched stars instead of `gl_PointSize`.

---

## 5. Galaxies

### 5.1 Mathematical Model

Each galaxy is a logarithmic spiral:

```
r(θ) = a · e^(b·θ)
```

Where:
- `a` = initial radius (varies per galaxy)
- `b` = tightness factor (0.2–0.4, controls spiral winding)
- `θ` = angle (0 to 4π for ~2 full turns)

**Star distribution:**
- 2–4 spiral arms per galaxy
- Stars placed along arms with gaussian noise (σ = arm width / 3)
- Density decreases with radius
- Central bulge: dense spherical distribution (r < 15% of galaxy radius)
- Halo stars: sparse random distribution (r < 120% of galaxy radius)

**Per-galaxy parameters:**

| Galaxy | Arms | Color (core → edge) | Star count | Radius | Tilt |
|--------|------|---------------------|------------|--------|------|
| Education | 2 | violet → lavender `#b478dc → #d4a8f0` | 800 | 1200 | 15° |
| Research | 3 | blue → cyan `#4a78cc → #7ab8e8` | 1000 | 1400 | -20° |
| Projects | 4 | orange → gold `#cc8844 → #e8c478` | 1200 | 1600 | 10° |
| Contact | 2 | green → teal `#44cc88 → #78e8b4` | 600 | 1000 | -10° |

### 5.2 Galaxy Rendering

**Stars:**
- `THREE.Points` with `BufferGeometry`
- Vertex attributes: position (xyz), color (rgb), size, brightness
- Custom shader:
  - Size attenuation by distance
  - Twinkling: `sin(time * star_speed + star_phase)` per star
  - Brightness falloff from arm center

**Gas/dust (nebula glow):**
- 6–10 `THREE.Sprite` per galaxy with additive blending
- Canvas-generated textures: layered radial gradients with noise
- Positioned along spiral arms
- Opacity fades with distance from camera (near: visible, far: faint glow)

**Core glow:**
- 1 large sprite at galaxy center
- Bright, slightly overexposed
- Size: ~20% of galaxy radius

### 5.3 Interactive Stars

Within each galaxy, certain stars are **content stars** (clickable). These are rendered as **separate `THREE.Mesh` objects** (not part of the background `Points` buffer) to enable reliable `Raycaster` hit detection.

**Implementation:**
- Each content star: `THREE.SphereGeometry(radius, 8, 8)` with emissive material
- Positioned at their spiral-arm coordinates (same math as background stars)
- Invisible `THREE.Mesh` hit sphere around each (radius: 20px screen-space desktop, 36px mobile) — recalculated on camera move via `vector.project(camera)` distance scaling
- Background `Points` do NOT include content star positions (no double-rendering)

**Visual distinction:**
- Slightly larger (1.5x) than background stars
- Subtle pulse animation (size oscillation via uniform)
- On hover (desktop): glow ring sprite appears, star label fades in (HTML overlay)
- Hit area: 20px radius (desktop), 36px radius (mobile)

**Content star data structure:**
```javascript
{
  id: 'hse-university',
  galaxy: 'education',
  position: { arm: 0, theta: 1.2, offset: 0.3 }, // along spiral
  label: 'HSE University',
  catalog: 'HD 34085 · β Orionis',
  title: 'HSE University',
  subtitle: 'Economics · Class of 2026',
  body: 'GPA 8.9 / 10\nRelevant coursework...',
  starType: 'Blue supergiant',
  starColor: [0.7, 0.8, 1.0],
  starSize: 1.2
}
```

---

## 6. Star Interaction Flow

### 6.1 Desktop

```
Hover star → label appears + glow ring
  ↓ click
Label fades (0.15s)
  ↓
HUD frame expands from center (clip-path animation, 0.35s)
  ↓
Typewriter text fills HUD
  ↓
"подробнее" button → zoom card slides in
  ↓
Zoom card: chamfer border, CRT effects, detailed content
  ↓
Close (X / click outside / Esc) → scroll resumes
```

### 6.2 Scroll Behavior During Interaction
- **Star click → scroll locked** (`overflow: hidden` on body or `preventDefault` on wheel)
- Scroll unlocked on card close
- Camera position preserved during lock

### 6.3 HUD Frame
- **Positioning:** HTML overlay element, positioned via `THREE.Vector3.project(camera)` to get screen coords of the clicked star. HUD placed offset from star (52px right, centered vertically). If near screen edge, flip to opposite side.
- Corner brackets (`┌ ┐ └ ┘`) with monospace text
- Expand animation: `clip-path: inset()` from center to full
- Content: catalog ID, star name, title, body text
- Typewriter effect at 30ms per character, single global cursor
- HUD does not need to track star movement (scroll is locked during interaction)

### 6.4 Zoom Card (Detail View)
- Chamfer clip-path border:
  ```css
  clip-path: polygon(
    var(--ch) 0%, calc(100% - var(--ch)) 0%,
    100% var(--ch), 100% calc(100% - var(--ch)),
    calc(100% - var(--ch)) 100%, var(--ch) 100%,
    0% calc(100% - var(--ch)), 0% var(--ch)
  );
  ```
- SVG border with `vector-effect: non-scaling-stroke`
- Tick marks and diagonal accent lines
- CRT effects: noise canvas, scanlines, chromatic aberration
- Staggered content reveal (title → subtitle → body → stats)

---

## 7. Navigation & UI

### 7.1 Depth Indicator
- Fixed right side, vertical progress bar
- 4 dots (one per galaxy) + connecting line
- Active dot highlights based on camera z-position
- Galaxy name label next to active dot
- Subtle, low opacity (0.1 default, 0.3 on hover)

### 7.2 Scroll Hint
- Inside galaxy (after 10s idle): gray text "scroll to continue ↓" at bottom center
- IBM Plex Mono 9px, 0.06 opacity
- Fades in, bob animation on arrow

### 7.3 Contacts
- Fixed bottom-right corner
- Appear after hero scrolls out of view
- TG + GH icons with monospace labels
- 0.12 opacity default, 0.45 on hover

---

## 8. Hyperspace Jump (End)

After Contact galaxy, continued scrolling leads to a dark void with the return jump prompt.

### 8.1 Trigger
- Scroll reaches 950–1000vh zone
- Single small text appears: `INITIATE RETURN JUMP` (IBM Plex Mono 10px, 0.08 opacity, pulsing)
- Click/tap to activate

### 8.2 Jump Sequence (exact steps)
1. **Lock scroll** — `document.body.style.overflow = 'hidden'`
2. **Star stretch ramp** — shader `u_stretch` uniform animates from 0 → 1 over 0.8s (ease-in)
3. **Max stretch hold** — stars are long streaks for 0.3s
4. **White flash** — CSS overlay `opacity: 0 → 1` over 0.1s
5. **Teleport** — during flash (screen is white): `camera.position.z = 0`, `window.scrollTo({top: 0, behavior: 'instant'})`
6. **Flash fade** — overlay `opacity: 1 → 0` over 0.3s
7. **Hero visible** — signal intro does NOT replay, hero is shown immediately with CRT effects active
8. **Unlock scroll** — `document.body.style.overflow = ''`

### 8.3 Mobile Considerations
- Touch scroll momentum must be killed before scroll reset: call `e.preventDefault()` on touchmove during jump
- `scrollTo({behavior: 'instant'})` avoids smooth-scroll interference

---

## 9. Performance Architecture

### 9.1 Rendering Pipeline
- **One master `requestAnimationFrame` loop** — no parallel rAF calls
- Loop updates:
  1. Camera position from scroll
  2. Star twinkling uniforms
  3. Galaxy sprite opacity (distance-based)
  4. CRT effects (hero only, gated by visibility)
  5. Three.js render call

### 9.2 Visibility Gating
- Hero CRT effects: stop when hero scrolled out of view
- Galaxy detail rendering: only for galaxies within camera frustum
- Star interaction raycasting: only for current galaxy zone
- Canvas noise: pause when not visible

### 9.3 GPU Budget
- Background stars: 2000 points (always rendered)
- Per galaxy: 600–1200 points + 6–10 sprites
- Only 1–2 galaxies visible at a time (frustum culling)
- Total simultaneous points: ~4000 max
- Target: 60fps on mid-range hardware

### 9.4 Memory
- Textures: canvas-generated (no image loads except fonts)
- Geometry: shared `BufferGeometry` per galaxy, no cloning
- Dispose galaxies that are far behind camera (optional, if memory is tight)

### 9.5 Scroll Handling
- Single `scroll` listener with `requestAnimationFrame` throttle
- No `getBoundingClientRect()` in hot paths
- Scroll position cached, updated once per frame
- Resize handler debounced (200ms)

---

## 10. Mobile Adaptation

### 10.1 Same Experience, Adapted
- Three.js flight preserved on mobile
- Touch scroll = flight (native scroll behavior)
- Tap on star = click interaction

### 10.2 Performance Adjustments
- Star count reduced to 60% (per galaxy and background)
- Gas/dust sprites reduced to 3–4 per galaxy
- Star stretch simplified (no radial effect, uniform stretch)
- Noise canvas disabled
- Hero CRT shift bands: reduced frequency (every 1–3s instead of 0.2–0.8s)

### 10.3 UI Adjustments
- No corner HUD on hero (not enough space)
- Star hit area: 36px radius (vs 20px desktop)
- Zoom card: full-screen overlay instead of centered modal
- Depth indicator: smaller dots, left side
- No hover states — tap directly opens star
- Scroll hint: "tap stars to explore" instead of click

### 10.4 GPU Fallback

**Detection criteria (any triggers fallback):**
- WebGL2 unavailable (`canvas.getContext('webgl2')` returns null)
- `renderer.capabilities.maxTextureSize < 4096`
- FPS probe: render 60 frames of background stars only; if average < 25fps → fallback
- `navigator.hardwareConcurrency < 2` (single-core device)

**Fallback experience:**
- Static sections with CSS parallax star background
- Each section shows galaxy name + clickable star list (HTML)
- Same content, same CRT card design, no Three.js flight
- Hero works normally (it's pure HTML/CSS/Canvas, no Three.js dependency)

---

## 11. Typography

| Element | Font | Size | Weight | Spacing |
|---------|------|------|--------|---------|
| Hero name | Saira | clamp(38px, 6.5vw, 76px) | 600 | clamp(6px, 1.5vw, 18px) |
| HUD text | IBM Plex Mono | 11px | 400 | 2px |
| Card title | Saira | 18px | 600 | 4px |
| Card body | IBM Plex Mono | 12px | 400 | 0.5px |
| Labels/catalog | IBM Plex Mono | 9–10px | 400 | 2–3px |
| Galaxy zone label | Saira | 14px | 500 | 6px |

All text uppercase. Colors: white at varying opacities (0.04–0.95).

---

## 12. Color System

```
Background:       #030508 (near-black with blue tint)
Text primary:     rgba(255, 255, 255, 0.95)
Text secondary:   rgba(255, 255, 255, 0.1)
Text ghost:       rgba(255, 255, 255, 0.04–0.07)

Galaxy: Education  #b478dc (violet)
Galaxy: Research   #4a78cc (blue)
Galaxy: Projects   #cc8844 (orange)
Galaxy: Contact    #44cc88 (green)

Star default:      rgba(255, 255, 255, 0.05–0.3)
Star interactive:  inherits galaxy color, brighter
CRT red channel:   rgba(255, 60, 60, 0.07–0.1)
CRT blue channel:  rgba(60, 100, 255, 0.07–0.1)
```

---

## 13. Content Map

### Galaxy 1: Education (violet)
| Star | Content |
|------|---------|
| Rigel (β Ori) | HSE University — Economics, 2022–2026, GPA 8.9 |
| Betelgeuse (α Ori) | School 21 (Sber) — Programming, 2023–2024 |

### Galaxy 2: Research (blue)
| Star | Content |
|------|---------|
| HD 168076 | ISSEK HSE — Research assistant |
| Vega (α Lyr) | VC market research |

### Galaxy 3: Projects (orange)
| Star | Content |
|------|---------|
| Eta Carinae | Landing Pipeline — AI landing page builder |
| Antares (α Sco) | Other projects TBD |

### Galaxy 4: Contact (green)
| Star | Content |
|------|---------|
| Proxima Centauri | Telegram, GitHub, Email, LinkedIn |

*Content stars and their real astronomical catalog IDs to be finalized during implementation.*

---

## 14. File Architecture

Single HTML file (`index.html`). JS organized as ES6 classes to manage complexity (~5000–7000 lines estimated).

```html
<!DOCTYPE html>
<html>
<head>
  <!-- Meta, fonts, CSS -->
  <style>
    /* 1. Reset & base */
    /* 2. Hero styles */
    /* 3. HUD frame */
    /* 4. Zoom card */
    /* 5. Navigation UI */
    /* 6. Mobile overrides (@media pointer:coarse, max-width) */
  </style>
</head>
<body>
  <!-- Hero HTML (signal intro, name, HUD corners) -->
  <!-- UI overlays (depth indicator, contacts, scroll hints) -->
  <!-- Hidden star content for a11y (see 15) -->
  <!-- Three.js canvas (injected by renderer) -->

  <script>
    // === CONFIG (all magic numbers, zone table, galaxy params) ===
    // === UTILS (debounce, lerp, smoothstep, spring) ===

    class AnimationCoordinator { /* single rAF, registers/unregisters systems */ }
    class HeroSystem { /* intro, CRT effects, reveal, visibility gating */ }
    class FlightSystem { /* scroll → camera z, velocity calc, zone detection */ }
    class GalaxyGenerator { /* spiral math, BufferGeometry, sprite generation */ }
    class StarInteraction { /* raycasting, HUD positioning, card open/close, scroll lock */ }
    class UISystem { /* depth bar, hints, contacts visibility */ }
    class HyperspaceSystem { /* end prompt, jump sequence, scroll reset */ }
    class MobileAdapter { /* touch handling, GPU probe, fallback trigger */ }
    class ErrorRecovery { /* WebGL context loss, shader fallback, font fallback */ }

    // === INIT (instantiate, wire up, start) ===
  </script>
</body>
</html>
```

---

## 14.1 Error Handling

| Failure | Detection | Recovery |
|---------|-----------|----------|
| WebGL context lost | `canvas.addEventListener('webglcontextlost')` | Pause rAF, show "Restoring..." overlay, attempt restore on `webglcontextrestored` |
| Shader compile fail | `gl.getShaderInfoLog()` returns errors | Fall back to `THREE.PointsMaterial` (no custom shaders) |
| Font load fail | `document.fonts.check()` returns false after 5s | CSS font-stack fallback: `'Saira' → 'Arial'`, `'IBM Plex Mono' → 'Courier New'` |
| rAF suspended | Tab backgrounded / battery saver | `document.addEventListener('visibilitychange')` — pause all animation, resume on return |
| Touch scroll conflicts on mobile | iOS rubber-band, Android fling | Use native scroll (no touch event hijacking). Scroll-to-z mapping handles momentum naturally. Non-linear mapping means overshoot in transit zones is harmless (just empty space). |

---

## 15. Accessibility

- `prefers-reduced-motion`: disable star stretch, CRT shifts, twinkling. Keep static content readable.
- `<main>` landmark wrapping content
- Heading hierarchy for screen readers (visually hidden if needed)
- **Keyboard navigation:** Hidden off-screen `<button>` elements for each content star, synced to star data. `Tab` cycles through buttons, `Enter` opens card. `aria-label` provides star name. Buttons are grouped per galaxy in a visually-hidden `<nav>` with `aria-label="Galaxy: Education"` etc.
- Skip-to-content link
- OG/Twitter meta tags for link previews
- **Contrast:** Informational text (card body, titles, subtitles) must meet 4.5:1. Decorative/ambient text (coordinates, catalog labels, corner HUD, scroll hints) at 0.04–0.1 opacity is explicitly exempt — these are atmospheric elements, not essential content. All essential content is accessible via star card interactions at full readable contrast.

---

## 16. Open Questions

1. Exact content text for each star card (body copy, stats, dates)
2. Additional projects to add to Galaxy 3
3. Whether to include a brief "about me" section within hero or as a separate star
4. Sound design — ambient space audio? (likely not for v1, but worth considering)
