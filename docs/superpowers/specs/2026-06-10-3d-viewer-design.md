# Olsano 3D Viewer — Design Spec
Date: 2026-06-10

## Overview

A standalone single-file HTML demo showcasing the `watering-can.glb` model with real-time material switching and albedo color picking. Built for the Olsano client (Italian kitchenware brand). No build tools, no dependencies to install — open `index.html` in a browser.

---

## Stack

| Concern | Choice |
|---|---|
| Output | Single `index.html` file |
| Renderer | Three.js r170+ `WebGPURenderer`, auto-fallback to `WebGLRenderer` |
| Model loading | `GLTFLoader` + `DRACOLoader` (CDN) |
| Lighting | `PMREMGenerator` + neutral studio `RoomEnvironment` |
| Controls | `OrbitControls` — orbit, zoom, pan. No auto-rotate. |
| Fonts | Google Fonts CDN: `Cormorant Garamond` (brand) + `Outfit` (UI) |
| Icons | Inline SVG only — no icon library |
| Imports | ES module importmap from `esm.sh` CDN (`three@0.170+`) |

---

## Layout

Full-bleed `100vw × 100dvh` canvas. No sidebars. Single floating pill control bar centered at the bottom.

```
┌─────────────────────────────────────────────────────┐
│  OLSANO                                 (top-left)  │
│                                                     │
│                   [3D Canvas]                       │
│               watering can model                    │
│              warm studio background                 │
│                                                     │
│        ╭─────────────────────────────╮             │
│        │ Matte Glass Polished Terra | ● │            │
│        ╰─────────────────────────────╯             │
└─────────────────────────────────────────────────────┘
```

Hint text (`Orbit · Zoom`) fades in on load, fades out after 4 seconds.

---

## Scene

### Background
Radial gradient painted on a `PlaneGeometry` background quad, or set via `scene.background = new THREE.Color(...)` with a gradient CSS on the canvas container:

```css
background: radial-gradient(ellipse at 40% 35%, #f0ebe3 0%, #e0d8ce 55%, #c8c0b4 100%);
```

### Lighting
- `PMREMGenerator` with `RoomEnvironment` for ambient PBR reflections
- `DirectionalLight` (intensity 1.5, warm white `#fff8f0`) from upper-right
- `AmbientLight` (intensity 0.4) for fill

### Camera
- `PerspectiveCamera` fov 45, positioned at `(0, 0.5, 3)` relative to model bounds
- `OrbitControls`: enable damping (`dampingFactor: 0.08`), no auto-rotate
- On model load: auto-center + auto-fit camera to bounding box

### Ground shadow
A Three.js `Mesh` (flat `CircleGeometry`) with a `MeshBasicMaterial` using a radial gradient canvas texture, placed just below the model on the Y axis. Not a CSS overlay — must be a scene object to stay aligned with the model.

---

## Materials

All use `THREE.MeshPhysicalMaterial`. The color (albedo) is the user-selected value except where locked.

| Preset | roughness | metalness | transmission | clearcoat | clearcoatRoughness | ior | notes |
|---|---|---|---|---|---|---|---|
| Matte Varnish | 0.85 | 0.0 | 0 | 0 | — | — | |
| Colored Glass | 0.05 | 0.0 | 0.92 | 0 | — | 1.5 | needs `transparent: true`, `side: THREE.DoubleSide` |
| Polished Varnish | 0.08 | 0.0 | 0 | 1.0 | 0.05 | — | |
| Terracotta | 0.90 | 0.0 | 0 | 0 | — | — | color locked to `#B5611A`, picker disabled |

On material switch: dispose old material, apply new `MeshPhysicalMaterial` to all meshes in the loaded GLTF scene graph (recursive traversal).

---

## Color Picker

- A single `<input type="color">` element, visually hidden, triggered by clicking the color dot in the pill.
- The dot displays the current color at all times.
- On `input` event: update `material.color` in real-time (no debounce needed for color inputs).
- 6 preset color swatches open in a small panel above the pill when the dot is clicked:
  - `#8B5E3C` (warm brown)
  - `#2C5F2E` (forest green)
  - `#1B4F72` (deep blue)
  - `#7B241C` (deep red)
  - `#F0E6D3` (cream/off-white)
  - `#2C2C2C` (near-black)
  - + "Custom" button that triggers `<input type="color">`
- When **Terracotta** material is active: color dot shows a lock icon overlay, preset panel is hidden, custom picker is disabled.

---

## UI Components

### Floating Pill
```
background: rgba(255, 255, 255, 0.88)
backdrop-filter: blur(20px)
border: 1px solid rgba(255, 255, 255, 0.9)
box-shadow: 0 8px 32px rgba(0,0,0,0.10), inset 0 1px 0 rgba(255,255,255,0.8)
border-radius: 100px
padding: 7px 10px
```

**Material tab — inactive:**
```
color: #666
padding: 5px 12px
border-radius: 100px
font: 500 11px/1 'Outfit'
```

**Material tab — active:**
```
background: #1a1a1a
color: #fff
```

Transition: `background 0.2s ease, color 0.2s ease`

**Divider:** `1px solid rgba(0,0,0,0.10)`, height 18px, margin 0 6px

**Color dot:** 22px circle, `border: 2.5px solid rgba(255,255,255,0.9)`, `box-shadow: 0 0 0 1.5px rgba(0,0,0,0.15)`

### Color Preset Panel
Appears above the pill on color dot click. Same glass morphism as pill. Contains 6 color swatches + custom button. Dismissed by clicking outside or selecting a color.

### Brand Label
Top-left, `position: absolute`. `Cormorant Garamond 500`, 15px, `letter-spacing: 0.18em`, `text-transform: uppercase`, color `rgba(60, 50, 40, 0.55)`.

### Hint Text
`"Orbit · Zoom"` — centered bottom, above pill, `Outfit 300` 10px, `color: rgba(60,50,40,0.35)`. Fades in at load, auto-fades after 4s via CSS animation.

---

## UI Color Palette

| Token | Value | Use |
|---|---|---|
| `--bg-warm-light` | `#F0EBE3` | Canvas gradient start |
| `--bg-warm-mid` | `#E0D8CE` | Canvas gradient mid |
| `--bg-warm-dark` | `#C8C0B4` | Canvas gradient edge |
| `--accent-sand` | `#C8B89A` | Decorative accents |
| `--text-primary` | `#1A1A1A` | Active pill tab bg |
| `--text-muted` | `#666666` | Inactive pill tabs |
| `--text-brand` | `rgba(60,50,40,0.55)` | Logo label |

---

## File Structure

```
demo-olsano/
├── index.html          ← entire app (HTML + CSS + JS inline)
└── .glb-file/
    └── watering-can.glb
```

No other files needed. The `.glb` path relative to `index.html` is `./.glb-file/watering-can.glb` (note dot-prefixed folder name).

---

## Interactions & States

| State | Behavior |
|---|---|
| Loading | Canvas background visible, pill hidden, centered spinner (thin ring, `#C8B89A`) |
| Load error | Centered inline message: `"Could not load model"`, muted style |
| Model loaded | Pill fades in (`opacity 0→1`, `translateY 8px→0`, 400ms ease-out) |
| Material switch | Instant (no transition animation needed) |
| Color change | Real-time, no lag |
| Terracotta selected | Color dot shows lock overlay SVG, preset panel suppressed |
| Hover on pill tab | Subtle `background: rgba(0,0,0,0.05)` |

---

## Accessibility & Constraints

- No emoji anywhere
- No Inter font
- No pure `#000000` — use `#1A1A1A`
- Full-height via `min-height: 100dvh` on canvas container, never `height: 100vh`
- `pointer-events: none` on all decorative elements (hint text, brand label, ground shadow)
- OrbitControls touch-enabled by default (mobile pinch-zoom)
