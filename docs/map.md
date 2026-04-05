# JavaScript World Map Control Specification (X-COM Style)

## 1. Overview

This document defines the specification for an interactive JavaScript-based world map control inspired by X-COM-style geoscapes. The control supports rendering a global map, zooming (scaling), dragging (panning), and selecting regions.

The goal is to provide a reusable, performant, and extensible component suitable for browser-based games and data visualization.

---

## 2. Core Features

* Render a 2D globe (orthographic projection)
* Smooth zoom (scale in/out)
* Drag/pan to rotate the globe
* Region selection (hover + click)
* Event-driven API
* High-performance rendering (Canvas 2D)

---

## 3. Coordinate System

### 3.1 Geographic Coordinates

* Longitude: `-180 .. +180`
* Latitude: `-90 .. +90`

### 3.2 Screen Coordinates

* Origin: top-left corner
* X axis: left → right
* Y axis: top → bottom

### 3.3 Projections

Three coordinate systems are in use:

**Equal-area space** — used for Voronoi cell computation. Lambert cylindrical equal-area projection prevents polar seeds from dominating the tessellation:

```
x = lon,  y = sin(lat · π/180)
bounds = [-180, -1, 180, 1]
```

Inverse: `lon = x,  lat = asin(y) · 180/π`.

After computing Voronoi in this space, cell vertices are inverse-projected back to lon/lat. Edges are then subdivided along great circle arcs (SLERP, ≤ 4° per segment) so they render correctly on the orthographic globe.

**Geographic space (lon/lat)** — the working format for cell polygons and hit testing after inverse projection from equal-area space.

**Display projection (orthographic)** — used for all canvas rendering. The globe is centred at `(lon0, lat0)` and projected to screen pixels:

```js
cosC = sin(lat0)·sin(lat) + cos(lat0)·cos(lat)·cos(lon - lon0)
// point is visible only when cosC > 0
x = cx + R · cos(lat) · sin(lon - lon0)
y = cy − R · (cos(lat0)·sin(lat) − sin(lat0)·cos(lat)·cos(lon - lon0))
```

`R` = globe radius in logical pixels (function of canvas size × zoom). `cx, cy` = canvas centre.

---

## 4. Rendering

### 4.1 Layers

1. Ocean sphere (filled circle)
2. Voronoi region fills (clipped to globe disc)
3. Region borders (Voronoi cell edges)
4. Overlays: labels, population, army/hero badges

### 4.2 Renderer

Canvas 2D. All draw calls use logical (CSS) pixels; the canvas physical size is CSS size × `devicePixelRatio`.

### 4.3 Resolution Handling

* Canvas physical size = CSS size × `devicePixelRatio`
* `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` applied once after resize; all draw calls use CSS pixels

---

## 5. Scaling (Zoom)

### 5.1 Scale Model

* Continuous zoom factor `zoom`
* `minScale = 0.5` (whole globe visible plus margins)
* `maxScale = 8.0`

Globe radius in pixels: `R = min(W, H) / 2 × zoom − 4`

### 5.2 Zoom Centering

Zoom is centred on the mouse position or viewport centre.

### 5.3 Input Methods

* Mouse wheel (`deltaY`)
* Trackpad / touch pinch (two-finger)
* Keyboard `+` / `-` (zooms on viewport centre)

### 5.4 Constraints

* Scale clamped to `[minScale, maxScale]`

---

## 6. Dragging (Panning)

### 6.1 Interaction

Dragging rotates the globe by adjusting `(lon0, lat0)` — the centre of the orthographic projection.

* Mouse: click + drag
* Touch: single-finger drag

### 6.2 Degrees per Pixel

```
dpp = 180 / (π · R)
lon0 -= dx · dpp
lat0 -= dy · dpp
```

### 6.3 Bounds

* `lon0` wraps freely; normalised to `[-180, 180]` after drag ends to prevent float drift
* `lat0` hard-clamped to `[-85, 85]`

---

## 7. Region Model

### 7.1 Region Seeds

Each geographic region is defined by a **center seed** `(lon, lat)` in degrees, stored in `map.json`. Seeds are derived from real-world bounding boxes as `((W+E)/2, (N+S)/2)` and are also hardcoded in `GEO_REGIONS` in `mapview.js` for use by the overlay renderer. The canonical source is `map.json`.

```js
GEO_REGIONS = {
  england_wales: { cx: -1.5, cy: 53 },
  france_north:  { cx:  2,   cy: 49.5 },
  // …
}
```

### 7.2 Delaunay Triangulation and Voronoi Cell Generation

Region polygons are **not hand-authored**. They are computed once at startup from the seed positions in `map.json` using spherical Delaunay triangulation. **No adjacency lists are stored in `map.json`** — adjacency is derived on-the-fly from the triangulation.

Every region (land, sea, and polar) has a seed `(lon, lat)` in `map.json`. Sea regions are included as seeds even though they are not rendered — their seeds act as bisector anchors that bound the cells of adjacent coastal land regions.

**Delaunay triangulation via 3-D convex hull.** For points on the unit sphere, the 3-D convex hull of those points is exactly the spherical Delaunay triangulation (Guibas & Stolfi 1985): every convex hull face is a Delaunay triangle, and every Delaunay edge is a hull edge.

The hull is built by randomised incremental insertion with conflict lists (de Berg et al. §11.2), expected **O(n log n)** time. Each uninserted point tracks a conflict list — the hull faces currently visible from it. On insertion: collect visible faces from the conflict list (BFS extends for numerical robustness), find the horizon, attach a cone of new faces, and bootstrap each new face's conflict list from the two faces that shared its base horizon edge (any point seeing the new face must have seen one of those two).

**Voronoi vertices from hull faces.** For each hull face (a, b, c) listed CCW from outside, the Voronoi vertex shared by the three adjacent cells is the outward unit normal `normalize((b−a)×(c−a))`. Equidistance: `N = (b−a)×(c−a)` is perpendicular to both `(b−a)` and `(c−a)`, giving `N·a = N·b = N·c` — equal dot-product with all three seeds, equal geodesic distance on the sphere.

**Adjacency.** Every hull edge `(i, j)` is a Delaunay edge; the adjacency graph is a free by-product of triangulation.

**Voronoi cells.** Each seed's Voronoi cell is the convex hull (on the sphere) of all valid vertices belonging to that seed. Vertices are sorted by angle around the seed's tangent frame, then each polygon edge is subdivided along the great circle arc connecting its endpoints (SLERP, at most 4° per segment) so edges follow the sphere surface rather than cutting through it when rendered.

**Polar regions.** The arctic (north of 75°N) and antarctica (south of 60°S) regions are explicit spherical-cap polygons: a sequence of points sampled every 2° of longitude along the bounding latitude circle, closed with the pole vertex. Dense sampling keeps each SLERP arc short enough (≈0.5°) to follow the latitude circle instead of bowing toward the equator.

### 7.2a Hemisphere Clipping

Before rendering, each cell polygon is clipped to the visible hemisphere using Sutherland-Hodgman:

* Vertices with `cosC > 0` are kept
* For each edge crossing the horizon (`cosC = 0`), the intersection point on the unit sphere is computed via linear interpolation in 3D Cartesian space and added to the output

This replaces the previous approach of simply filtering out invisible vertices (which broke polygon shapes for large cells).

### 7.3 Hit Detection

Point-in-polygon (ray casting) against Voronoi cell polygons in lon/lat space:

1. Inverse-project screen click `(sx, sy)` to `(lon, lat)` via inverse orthographic
2. Test `(lon, lat)` against each cell polygon
3. Return the matching region

Because Voronoi cells partition the plane completely, every visible point on the globe maps to exactly one region.

### 7.4 Spatial Lookup

`_allGeo` — ordered list of `{ key, poly, cLon, cLat }` for all GEO_REGIONS entries.
`_byKey` — map from geo key → world region entry (for O(1) game-data lookup after hit).
`_entries` — world regions that matched a geo seed (used for overlays and `selectById`).

---

## 8. Region Selection

### 8.1 Interaction States

* `hoveredRegion` — updated on `mousemove`
* `selectedRegion` — set on `click` or `selectById(id)`

### 8.2 Visual Feedback

* Hover: brighter fill + highlighted border (`COL_BORDER_HOV`)
* Selected: glowing shadow (faction colour) + thicker border (`COL_BORDER_SEL`)

### 8.3 View Modes

The fill colour of each region changes with the active view mode:

| Mode | Fill |
|---|---|
| `political` | Faction colour (dimmed unless hovered) |
| `population` | Blue heat map (log scale, 0 → 1400M) |
| `unrest` | Green → amber → red (0 → 80) |
| `prosperity` | Red → amber → green (0 → 100) |

### 8.4 Callback

```js
new MapView(canvas, (region) => { /* region or null on deselect */ })
```

---

## 9. Overlays

Drawn after region fills, at each region's projected center:

* **Region name** — 10 px label
* **Population** — 8 px sub-label (`XXM` or `X.XB`)
* **Unrest dot** — coloured circle above label if `unrest > 0`
* **Army badge** — `⚔N` if armies present
* **Hero badge** — `★N` if heroes present

---

## 10. Performance

* Voronoi computed once at module load; cells stored in memory
* Region polygons projected fresh each frame (orthographic depends on `lon0/lat0`)
* `requestAnimationFrame` not used explicitly — redraws triggered by input events only
* `ResizeObserver` triggers `_layout()` (recomputes projected coords + redraws) on container resize

---

## 11. API

### 11.1 Constructor

```js
new MapView(canvasEl, onSelectCallback)
```

### 11.2 Methods

| Method | Description |
|---|---|
| `render(world)` | Load world data and draw |
| `setView(mode)` | Switch map view mode |
| `selectById(regionId)` | Select region by game ID, rotate globe to it |
| `deselect()` | Clear selection |

---

## 12. Mobile Support

* Pinch-to-zoom (two-touch)
* Single-finger drag to rotate
* Tap to select region

---

## 13. Optional Enhancements

* Day/night terminator shading
* Animated radar sweep (X-COM style)
* Markers (armies, heroes)
* Time acceleration controls
* Weighted Voronoi (geographic cost map biases cell shapes toward natural borders)

---

## 14. Summary

The map control renders a rotatable orthographic globe. Geographic regions are partitioned using a Voronoi diagram computed from seed centers, giving gapless coverage with no hand-authored polygon data. All rendering, hit testing, and selection work against these computed cells.
