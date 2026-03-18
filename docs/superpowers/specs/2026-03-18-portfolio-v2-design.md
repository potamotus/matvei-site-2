# Portfolio v2 — Design Specification

**Author:** Matvei Tkhai + Claude
**Date:** 2026-03-18
**Status:** Draft
**Repo:** https://github.com/potamotus/matvei-site-2

---

## 1. Overview

A single-page interactive portfolio site built as a **scroll-driven flight through space**. The visitor travels through four spiral galaxies — each representing a life domain (Education, Research, Projects, Contact). Stars within galaxies are clickable and reveal content in sci-fi HUD frames with CRT effects.

**Tech stack:** Single HTML file, Three.js (WebGL), vanilla JS, no build tools.

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

### 3.1 Signal Intro
- Full-screen dark overlay with noise canvas
- 3 lines typed at 25ms per character:
  1. `SCANNING DEEP SPACE...`
  2. `SIGNAL ACQUIRED · 1420.405 MHz`
  3. `LINK ESTABLISHED`
- Noise intensity decreases with each line
- Click anywhere to skip
- Duration: ~3 seconds total

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
- Three.js `PerspectiveCamera`, FOV 60, near 0.1, far 50000
- Camera moves along z-axis based on scroll position
- Scroll position → camera z mapping is non-linear:
  - **Inside galaxy:** slow mapping (1vh scroll ≈ small z change) — linger and explore
  - **Between galaxies:** fast mapping (1vh scroll ≈ large z change) — acceleration zone

### 4.2 Scroll Zones

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

Within each galaxy, certain stars are **content stars** (clickable):

**Visual distinction:**
- Slightly larger (1.5x) than background stars
- Subtle pulse animation (size oscillation)
- On hover (desktop): glow ring appears, star label fades in
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
- Corner brackets (`┌ ┐ └ ┘`) with monospace text
- Expand animation: `clip-path: inset()` from center to full
- Content: catalog ID, star name, title, body text
- Typewriter effect at 30ms per character, single global cursor

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

After Contact galaxy, continued scrolling leads to:

1. **Dark screen** — nearly black, single small text: `INITIATE RETURN JUMP` (clickable)
2. **Click →** star stretch animation intensifies (stars become long streaks)
3. **Flash** — brief white flash (0.1s)
4. **Back to hero** — scroll position resets to 0, hero is visible again
5. Signal intro does NOT replay — hero appears immediately

Implementation: CSS animation for star streaks + JS scroll reset + hero state management.

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
- Detect weak GPU via `renderer.capabilities` (max texture size, vertex count)
- Fallback: static sections with parallax star background (CSS-based)
- Each section shows galaxy as static image + clickable star list
- Same content, same CRT card design, no Three.js flight

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

Single HTML file (`index.html`) with internal structure:

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
    /* 6. Mobile overrides */
  </style>
</head>
<body>
  <!-- Hero HTML (signal intro, name, HUD corners) -->
  <!-- UI overlays (depth indicator, contacts, scroll hints) -->
  <!-- Three.js canvas (injected by renderer) -->

  <script>
    // === CONFIG ===
    // === UTILS (debounce, spring, lerp) ===
    // === AnimationCoordinator (single rAF master loop) ===
    // === HeroSystem (intro, CRT effects, reveal) ===
    // === FlightSystem (scroll → camera, velocity, zones) ===
    // === GalaxyGenerator (spiral math, star placement) ===
    // === StarInteraction (raycasting, HUD, cards) ===
    // === UISystem (depth bar, hints, contacts) ===
    // === HyperspaceSystem (end jump, reset) ===
    // === MobileAdapter (touch, GPU detect, fallback) ===
    // === INIT ===
  </script>
</body>
</html>
```

---

## 15. Accessibility

- `prefers-reduced-motion`: disable star stretch, CRT shifts, twinkling. Keep static content readable.
- `<main>` landmark wrapping content
- Heading hierarchy for screen readers (visually hidden if needed)
- Star content accessible via keyboard (Tab to navigate stars, Enter to open)
- Skip-to-content link
- OG/Twitter meta tags for link previews
- Minimum text contrast: 4.5:1 for body text, 3:1 for decorative labels

---

## 16. Open Questions

1. Exact content text for each star card (body copy, stats, dates)
2. Additional projects to add to Galaxy 3
3. Whether to include a brief "about me" section within hero or as a separate star
4. Sound design — ambient space audio? (likely not for v1, but worth considering)
5. Loading screen design while Three.js initializes
