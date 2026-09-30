# Quaternion Graph

**A fast, dependency-free 3D graph of a note vault — rendered on a sphere, rotated with quaternions, and fingerprinted with Bell numbers.**

[![Live demo](https://img.shields.io/badge/demo-sololearn-blue)](https://sololearn.com/compiler-playground/WY1cw1AhM2De/?ref=app)
[![No dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)]()

---

## What it is

A single HTML file that renders a hierarchical vault — folders and their files — as a **clustered 3D sphere**. Folders sit on a Fibonacci-distributed shell, and each folder's files spiral outward in a tangent-plane neighbourhood. The whole scene can be dragged, pinched, and zoomed like a physical object.

It is **not a plugin** and not integrated with any note-taking app. It generates a synthetic vault from a seeded PRNG and visualizes it. Think of it as a **design study and technical demo**, inspired by Obsidian's graph view, exploring what a quaternion-driven spherical layout could feel like.

No build step, no bundler, no framework, no WebGL. Just a `<canvas>`, some math, and a lot of care about the hot loop.

---

## Features

### Rendering
- **Canvas 2D renderer** — no WebGL, no Three.js, no D3. Ships as one file.
- **Zero per-frame allocations** — positions, projections, and edge indices live in pre-allocated typed arrays.
- **Depth-bucketed edges** — thousands of links collapse into 16 `ctx.stroke()` calls.
- **Inlined quaternion rotation** — the Rodrigues formula is expanded directly inside the projection loop.
- **DPR-aware** — crisp on Retina and high-DPI phones, capped at 2×.
- **Adaptive label budget** — above ~500 nodes, only the closest labels draw.

### Interaction
- **Quaternion camera** — drag converts pointer deltas into axis–angle rotations composed by left-multiplication. No gimbal lock.
- **SLERP on reset** — "Reset view" interpolates along the shortest geodesic on *S³*.
- **Inertial spin** — release a drag and the sphere keeps rotating with exponential decay.
- **Unified pointer input** — mouse, touch, and pen via the Pointer Events API.
- **Hover & select** — highlights a node's neighbours and edges; tap background to deselect.
- **Keyboard shortcuts** — `+`/`-` zoom, `0` reset, `Space` auto-rotate, `C` color, `S` shuffle, `F` equations, `Esc` deselect.

### Combinatorial signature
- **Bell number HUD** — displays *Bₙ*, the number of ways to partition *n* nodes into non-empty groups.
- Exact via **Aitken's triangle** (BigInt, up to *n* = 120), asymptotic beyond that via **Moser–Wyman** with a Halley-iterated **Lambert W**.

### Equations panel
- **13 formulas documented** — Hamilton product, axis–angle construction, normalization, Rodrigues rotation, SLERP, Bell triangle, Moser–Wyman, Lambert W, Fibonacci sphere, tangent-plane basis, spherical-cap placement, perspective projection, depth cue.
- Rendered with **MathJax 3** (SVG output).
- **Storage-hardened** — installs an in-memory `localStorage` shim before MathJax loads, so it works in sandboxed iframes (SoloLearn, CodePen, etc.).

### Responsiveness
- **Mobile-first CSS** with `clamp()` fluid typography.
- **Safe-area insets** for notched phones.
- **`100dvh`** for correct mobile viewport height.
- **`ResizeObserver`** on the canvas — adapts to split-screen, rotation, any layout change.

### Performance
- **Render loop pauses** when the tab is hidden.
- **rAF-batched wheel zoom** and slider rebuilds.
- **Cached font strings** — no re-parsing font shorthand per frame.
- **Zero allocations in the tick** — no array literals, object literals, or closures per frame.

---

## Quick start

Open `index.html` in any modern browser. That's it.

```bash
git clone https://github.com/<you>/quaternion-graph.git
cd quaternion-graph
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

To serve it locally (some browsers restrict file://):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

---

How it works

1. Rotation

The camera is a unit quaternion q ∈ S³, not Euler angles. Pointer deltas become an axis–angle rotation:

```
q' = q_axisAngle(-dy, dx, 0, ||Δ|| · k) · q
```

Because quaternions compose by multiplication, successive drags accumulate cleanly. Because S³ is a double cover of SO(3), there is no gimbal lock and no singularity at the poles.

Rotating a node's world position uses the sandwich product v' = q v q*, inlined in the projection loop as:

```js
t  = 2 * (q_v × v)
v' = v + q_w · t + q_v × t
```

~15 multiplies instead of the 28 required by the naïve form.

2. Layout

· Folders are placed on a Fibonacci sphere using the golden angle φ = π(3 − √5) ≈ 137.5°. Near-uniform spacing for any folder count — the same trick sunflowers use to pack seeds.
· Files spiral outward from their folder in the folder's tangent plane:
  ```
  n = d·cos(α) + t₁·sin(α)·cos(φ_rank) + t₂·sin(α)·sin(φ_rank)
  ```
  α grows with rank, φ_rank advances by the golden angle. The result is a compact, evenly-filled cluster on a spherical cap, re-projected onto the sphere.
· Sphere radius grows sub-linearly: R = 80 + 13·√N. Individual nodes appear smaller as the vault grows — the sphere fills the same screen space regardless of node count.

3. Projection

A pinhole camera at distance C·R along the view axis:

```
p_screen = (x·s, y·s),  s = (C·R / (C·R − z)) · f

f = 0.43 · min(W, H) / (R · C/(C−1))
```

The C/(C−1) term pre-compensates for near-side foreshortening, so the sphere fills ~86% of the viewport's shorter axis.

4. Rendering

Edges are bucketed into 16 depth slices, each stroked once with per-bucket opacity and stroke width. Thousands of stroke calls collapse to 16.

Nodes are depth-sorted back-to-front, drawn as filled arcs. Opacity follows a quadratic depth cue:

```
α(z) = 0.18 + 0.82 · (½t² + ½t),  t = (z + R) / (2R)
```

Labels are drawn in the same order, budgeted above 500 nodes.

---

The Bell number HUD

For n nodes, the HUD shows Bₙ — the number of ways to partition an n-element set into non-empty subsets.

n Bₙ
1 1
5 52
10 115,975
20 5.17 × 10¹³
50 1.86 × 10⁴⁷
100 4.76 × 10¹¹⁵

Two algorithms share the work:

Exact (n ≤ 120) — Aitken's triangle, using BigInt:

```
A(n, 0) = A(n−1, n−1)
A(n, k) = A(n, k−1) + A(n−1, k−1)
Bₙ = A(n, 0)
```

Asymptotic (n > 120) — Moser–Wyman:

```
Bₙ ~ (1/√n) · (n/W(n))^(n + ½) · exp(n/W(n) − n − 1)
```

where W is the Lambert W function, solved via Halley's method (cubic-convergent variant of Newton's). Three or four iterations give machine precision.

---

Project structure

```
.
├── index.html          # everything: markup, styles, math, renderer
└── README.md
```

Single-file by design. No build step, no node_modules, no framework version to fight. Drop it on any static host — GitHub Pages, Netlify, a USB stick — and it runs.

Inside index.html:

Section Responsibility
1. Quaternion math qMul, qFromAxisAngle, qNormalize, qRotate, slerp
2. Bell numbers BigInt triangle, Lambert W, asymptotic fallback
3. State module-level variables, no globals leaked
4. Vault generation synthetic folders/files/edges
5. Layout Fibonacci sphere + tangent-plane spiral
6. Canvas sizing DPR-aware ResizeObserver
7. Projection world → screen, inlined rotation
8. Render edges, nodes, labels, depth sort
9. Palette color mode + monochrome
10. Regenerate rebuild pipeline for slider changes
11. Interaction pointer, wheel, pinch, keyboard
12. Refresh active hover/select neighbourhood
13. Equations panel MathJax rendering + accordion
14. Loop rAF tick with delta-time compensation
15. Boot resize, initial build, start

---

Customization

Colors are CSS custom properties on :root:

```css
:root {
  --bg: #08080a;
  --fg: #e0e0e8;
  --accent: #c5b6ff;
  --panel: rgba(15, 15, 20, .85);
  /* ... */
}
```

Folder colors live in the FOLDER_COLORS array — 30 hand-picked hues that cycle. Other tunables:

```js
R = 80 + 13 * Math.sqrt(N);          // sphere radius
n.r = 5.5 + Math.min(3.5, d * 0.30); // folder node radius
EDGE_OP[b] = 0.02 + t * 0.12;        // edge opacity per bucket
```

Auto-rotate speed: the 0.0016 in the tick. Camera distance: CAM = 4. Zoom limits: ZOOM_MIN / ZOOM_MAX.

---

Known limitations

· Synthetic data only. There is no file reader, no vault import, no real-world integration. The vault is generated from a seeded PRNG.
· No persistence. Seed, orientation, and zoom reset on reload.
· Canvas 2D ceiling. Comfortably handles a few thousand nodes at 60 FPS. For 10,000+, either reduce edge detail or port to WebGL.
· localStorage in sandboxed iframes. SoloLearn and similar playgrounds block storage access, which breaks MathJax's font cache. The code installs a shim, but be aware if you embed it elsewhere.

---

Possible future directions

Nothing here is committed work — just ideas that have come up:

· Obsidian plugin. The community already has several 3D graph plugins (Armillary, Vault Orrery, Galaxy Graph, 3D Graph), but none use a quaternion camera or a deterministic spherical layout. A port would mean replacing the synthetic vault generator with app.vault.getMarkdownFiles() and app.metadataCache.resolvedLinks, and moving the renderer into an ItemView. Rough sketch: 1–2 weeks for someone familiar with the Obsidian API.
· WebGL renderer for the 10k+ node case.
· Vault importers for other note tools (Roam, Logseq, Bear).
· More equations documented in the panel.

If any of these get built, they'll be linked here.

---

Inspiration

· Obsidian — the graph view that started it
· Hamilton, Rodrigues, and Cayley — for the quaternion algebra
· Eric Temple Bell — for the numbers
· Moser & Wyman (1955) — for the asymptotic
· Sunflowers — for the golden angle

---

Contributing

Issues and PRs welcome. Particularly interested in:

· Performance reports on large vaults
· WebGL ports
· Additional documented equations
· Bug reports on mobile (iOS Safari especially)

Open an issue or tag me on GitHub.
