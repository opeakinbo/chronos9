# Chronos9 — Globe Landing Page

## Task

Build a **marketing landing page for Chronos9** centered on a 3D particle globe. The globe is a `THREE.Points` object — thousands of particles distributed across a sphere surface, rendered via WebGL. Each particle occupies a precise geographic location on Earth, forming an accurate world map. The globe should feel like a scientific instrument — refined, neutral, and observational. The surrounding page is a modern SaaS marketing site: top nav, hero text with cycling accent word, CTA, and a trust-marker logo carousel.

---

## Page Structure

```
┌─────────────────────────────────────────────────────────────────┐
│ [Chronos9]     Products  Pricing  Blog     Sign up  [☀][⚙]    │  ← Nav: logo left, links centered, sign-up right
│                                                                 │
│                                                                 │
│       🌐               AI agents for                            │
│                        real estate                              │  ← Globe left / hero right, both vertically centered
│                                                                 │
│   Meet the workforce of the next                                │
│   decade. We build hyper-                                       │
│   intelligent, specialized AI…                                  │
│                                                                 │
│   [ Get started ]                                               │
│                                                                 │
│                                                                 │
│      Google   Amazon   Stripe   Microsoft   Vercel   …         │  ← Centered monotone logo carousel
└─────────────────────────────────────────────────────────────────┘
```

### Layout rules
- Page uses a **100px left/right margin** on desktop (`.page-stack` padding)
- **Nav** is a 3-column grid: logo left, nav links centered, sign-up right
- **Hero row** is a 2-column grid (`1fr 1fr`) that fills the vertical space: **globe on the left, hero content on the right**, both **vertically centered** (`align-items: center`)
- **Hero text (title, description) is left-aligned**; CTA sits flush-left below
- The globe canvas is fixed to the viewport, and `rotGroup.position.x` is offset to the **left** so the sphere visually anchors to the center of the left column (respecting the 100px page margin). On mobile (< 768px) the offset resets to 0 and the layout collapses to a single centered column (globe slot hidden; globe centered behind text)
- **Logo carousel** is centered (`max-width: 1200px`) with mask-fade edges, pinned to the bottom (`margin-top: auto` via flex)

---

## Setup

- Use `THREE.WebGLRenderer` with `antialias: true`
- Single `THREE.PerspectiveCamera` at ~45° FOV, positioned at z=12
- Wrap all particles in a `THREE.Group` — rotate the group, not the scene
- Use `THREE.BufferGeometry` with `position` (Float32Array) and `color` (Float32Array, 4 components: RGBA) attributes
- Use `THREE.PointsMaterial` with `vertexColors: true`, `transparent: true`, `sizeAttenuation: true`, `depthWrite: false`
- Canvas is `position: fixed; inset: 0; z-index: 0` — the page content sits on top with `z-index: 10+`

---

## Particle Distribution & World Map

- Generate 400–100000 particles (default 50000) using **Fibonacci sphere / golden angle** distribution for even surface coverage
- Each particle represents a specific latitude/longitude coordinate on Earth
- **Land vs. ocean classification**: For each particle, compute its lat/lon from its 3D position and test against the land polygon dataset using point-in-polygon ray casting
- Store base 3D positions on the unit sphere scaled by `globeRadius`
- Per frame, derive depth from each particle's world-space Z after group rotation to compute color and opacity
- **World map data**: Real geographic coordinates from **Natural Earth 50m** (higher fidelity than the prior 110m), Douglas-Peucker simplified at ~0.15° tolerance. Yields ~723 polygon rings and ~9,148 vertices total. Rings with antimeridian-crossing edges are split so no artificial seam appears at ±180° longitude. Tiny islands (area < 0.03 sq°) are omitted to control file size (~117 KB of polygon data inline)
- **Broad-phase acceleration**: Each ring's bounding box is computed once at load time into a typed array. `isLand()` skips any ring whose bbox doesn't contain the query point — roughly 8× faster than scanning every ring for every particle, keeping `buildGlobe()` responsive at 100k particles
- **Data integrity**: Programmatically derived from `world-atlas/land-50m.json` (jsdelivr CDN) by expanding TopoJSON arcs, simplifying via Douglas-Peucker, splitting antimeridian-crossing edges, and rounding to 2 decimal places. **Map orientation check**: Longitude calculated as `Math.atan2(-z, x)` to ensure correct E/W orientation. Latitude range is -90 to +90 (south to north). No artificial latitude or meridian lines should appear on the globe

---

## Color & Depth

Particles closer to camera (positive Z in world space) appear brighter and more opaque; farther particles are dimmer and more transparent.

**Dark mode**
- Background: `#0B0C10` (near-black)
- Land particles near → far: bright gray → dark gray, opacity 1.0 → 0.5
- Ocean particles near → far: medium → very dark gray, opacity 0.6 → 0.15

**Light mode**
- Background: `#F5F2EC` (warm off-white)
- **Continent (land) color targets `#252525`** (dark slate) — near continents render at ~#252525, far ones a lighter shade
- Ocean particles are a lighter shade of the continent color (gray-brown), so the map reads as darker landmasses against a subtly-tinted ocean

The color curves live in `COLOR_TABLE` (`globe.js`), indexed by `dark × land`.

---

## Interactions

### Auto-Rotation
- Base angular velocity `0.0003 × speed` per frame on `rotGroup.rotation.y`
- Speed slider 0–10; 0 pauses

### Drag + Inertia
- Pointer Events with `setPointerCapture` so release always fires on canvas
- `smoothVelX` is an EMA of per-frame velocity; clamped to ±0.06 on release
- No snap-back — globe stays where released
- Each frame: `rotation.y += velX; velX *= damping`

### Hover — Light + Lift
- Ray-cast from cursor to the globe sphere
- All particles within `HOVER_RANGE = 1.4` of the hit point are illuminated
- `hoveredParticles: Map<index, weight>` where `weight = 1 − dist/HOVER_RANGE` (squared in `updateColors` for an ease-out gradient)
- **Gentle radial lift**: every hovered particle is also pushed outward from the globe center by `globeRadius × 0.035 × weight²` — a subtle bump that makes the hovered area feel tactile, like a light indenting a soft surface
- Offsets ease toward the lift target each frame via frame-rate-independent exponential blend (`k = 1 − exp(−dt × 8)`)
- **Perf optimization**: `hoverDirty: Set<index>` tracks particles with nonzero hover offsets. The per-frame ease loop iterates only this set (not all N particles) and self-prunes when offsets reach zero — so an idle globe does zero hover work per frame

### Click Ripple — Wave Propagation
- On click (drag distance < 5px), spawn a `wave` with its local-space hit point and a 1.4s lifetime
- Each frame, the wave expands outward at `waveSpeed = R × 1.6` units/second
- Particles within a Gaussian ring (`sigma = R × 0.35`, so soft on both sides) receive an impulse pushing them radially outward from the globe center
- Impulse strength falls off as `exp(−delta² / 2σ²)` where delta is each particle's distance from the wave's current ring radius — a **soft edge**, not a hard boundary
- Wave amplitude decays linearly over its lifetime
- **Spring return**: every impulsed particle has a per-particle velocity and a spring-back term (`spring = 12, damp ≈ 0.08^dt`). The motion feels like a drop of water rippling across the surface — not a shove-and-snap
- Ripple and hover share the same `offsets` buffer; when a ripple is active for a particle, hover easing is suspended on that particle until the ripple timer expires

---

## Controls Panel (top-right)

- Fixed panel at `top: 24px; right: 24px`, 250px wide
- Glass look: translucent background (`backdrop-filter: blur(12px)`), thin border, 10px radius
- Header with "GLOBE CONTROLS" label and a **collapse/expand button**
  - 26×26 square hit target (generous click region, not a tiny 16×16 icon)
  - SVG `−` when open, `+` when collapsed
  - Default / hover (lighter background) / pressed (darker background) states
- When collapsed: `.panel-body` height smoothly animates to 0 (max-height transition) and its margin drops, so there's no redundant empty space below the header
- Controls:

| Control | Range | Default |
|---|---|---|
| Particles | 400 – 100000 | 50000 |
| Radius | 1.0 – 5.0 | 3.0 |
| Speed | 0 – 10 | 3 |
| Size | 0.001 – 0.30 | 0.005 |
| Damping | 0.80 – 0.99 | 0.93 |
| Shape | ● / ■ / ▲ | ● |

### Debounced Rebuild
Particle-count and radius changes trigger a full `buildGlobe()` (re-allocate typed arrays, re-seed Fibonacci sphere, re-classify land). This is expensive at 100k particles. `scheduleRebuild()` debounces by 120 ms so dragging a slider doesn't rebuild on every `input` event — only after the user pauses. All other controls (speed, size, damping, shape) update state live.

---

## Dedicated Theme Toggle

- A separate floating button (`#theme-control`) sits just to the left of the controls panel
- Circular-feel square button (40×40, 10px radius) with sun icon in dark mode, moon icon in light mode
- Same glass style as the panel
- Switches `body.dark` ⇄ `body.light`, which swaps every CSS custom property and the Three.js clear color in one shot

---

## Navigation Bar

- Fixed at top, `padding: 22px 40px`
- Left: **Chronos9** logo (DM Sans 700, `#252525` in light / `#EDEDED` in dark, 20px) + nav links (Products, Pricing, Blog)
- Right: Sign up link
- On mobile (< 768px): nav collapses behind a hamburger button; tapping it slides a glass sheet down with the links stacked vertically
- `pointer-events: none` on the nav container with children set to `auto`, so the globe remains draggable in the unused gaps

---

## Hero

- Left-aligned column, max 560px wide, `padding-top: 160px`
- **Headline**: "AI agents for" on line 1, then a cycling word on line 2 — `real estate`, `stock markets`, `governments`, `education`, `music`, `everyone`. Cycles every 2.2 s with a 260 ms fade-out / fade-in cross
- **"AI agents for"** is the primary hero color (`#252525` light / `#F2F2F2` dark). The **cycling word** is the brand accent `#ff4c00` in both themes
- **Description**: secondary shade of the hero color (`#5C5C5A` light / `#A8A8A8` dark)
- **CTA button**: "Get started" in `#ff4c00` background, white text, 14×28 padding, 8px radius. Hover → `#ff6528`

---

## Logo Carousel

- Sits at the **bottom of the page stack**, horizontally centered, `max-width: 1200px`
- Horizontal infinite-scroll: `.logo-track` contains each wordmark twice and translates from 0 → −50% over 40 s, producing a seamless loop
- Logos are **monotone wordmarks** (plain text: Google, Amazon, Stripe, Microsoft, Vercel, GitHub, Coinbase, DoorDash) — they inherit `--logo-color`, which swaps with theme (soft gray in both modes). No SVGs, so nothing breaks across browsers or themes.
- A left/right edge fade is applied via `mask-image: linear-gradient(...)` so the logos feather into the background rather than pop in

---

## Typography & Global Styles

- **Font**: `DM Sans` from Google Fonts (weights 400, 500, 600, 700)
- `-webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale;` applied globally
- CSS custom properties drive the theme; `body.dark` / `body.light` toggles redefine them and every color-bound rule transitions in 0.4 s

---

## Mobile Responsiveness

Breakpoints at **768px** and **480px**:

- **< 768**: nav collapses to hamburger; hero padding reduces (120px top, 20px sides); hero title scales down; controls panel moves to bottom-left/right and spans the viewport; logo carousel tightens
- **< 480**: logo size trimmed; hero top padding further reduced
- Globe canvas always fills the viewport — the layout is stacked, not side-by-side, on narrow screens

---

## Technical Notes

- Three files: `index.html`, `style.css`, `globe.js`
- **Three.js** v0.184.0 loaded via `importmap` from `cdn.jsdelivr.net`
- `globe.js` is loaded with `<script type="module">`
- No `OrbitControls` — drag + inertia are implemented directly for full control over feel
- `window resize` updates `renderer.setSize` and camera aspect
- Color buffer is updated every frame (depth-based)
- Position buffer is only committed when a wave, ripple, or hover offset changed — idle frames skip the `commitPositions` pass
- **World map data is embedded inline** in `globe.js` (~117 KB of polygon rings) — no separate data file to fetch
- **Particle shapes**: rendered via a 64×64 canvas texture (circle / square / triangle) with `alphaTest: 0.05`

---

## File Structure

```
globe/
├── index.html   — nav, hero, carousel, controls panel, theme toggle, canvas
├── style.css    — DM Sans, custom properties, all theme transitions, responsive breakpoints
└── globe.js     — all Three.js logic, world map data (50m), interactions, UI bindings
```

---

## Quality Assurance Checklist

- **Map orientation**: Africa, Australia, and all continents display with correct orientation (not inverted or mirrored). Africa should sit between Europe and Asia with the Cape pointing south
- **Seam detection**: No artificial lines on the globe. Specifically:
  - Antimeridian (±180° longitude) discontinuities — should be seamless (rings are split at load time)
  - No stray horizontal or vertical lines that don't correspond to real geography
  - If stray lines reappear, check `LAND_POLYGONS` for antimeridian-crossing edges
- **Interaction feel**:
  - Click ripple should look like a drop of water: a soft expanding ring with gaussian falloff, not a hard circle
  - Hover should feel like a gentle light with the particles subtly lifting — not jittering or snapping
  - Dragging should coast with inertia and never snap back
- **Performance**:
  - Increasing particle count via the slider should not freeze the UI — the rebuild is debounced 120 ms
  - Idle hover (no cursor over globe) should contribute zero work per frame
- **Responsive**:
  - At < 768px the hamburger appears and the panel repositions to the bottom
  - Text remains readable (no overflow) from 320px up
- **Themes**: Both light and dark modes should be visually coherent — the globe, nav, hero, panel, carousel, and theme toggle all swap in lockstep via CSS custom properties

---

## Deliverable

A landing page that feels like a premium AI product site: precise, confident, slightly futuristic — with a scientifically-grounded globe as its hero, not as a gimmick.
