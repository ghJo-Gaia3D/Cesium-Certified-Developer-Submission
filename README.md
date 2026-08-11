# Cesium Certified Developer — Project Submission

**Geonhee Jo** · GAIA3D, Inc. · ghjo@gaia3d.com · Cesium ion: `ghJo-Gaia3D`

---

## The project

**KICT Urban Flood Digital Twin** — a CesiumJS web application built for KICT
(Korea Institute of Civil Engineering and Building Technology) to support flood
mitigation planning *before* construction. A planner draws a candidate facility
directly on the terrain — a detention tank, an open retention basin, a floodwall,
a drainage grate — runs a cellular-automata inundation analysis, and sees how the
flood footprint and building-level risk change.

The requirement that shaped the architecture: **the terrain the user edits must be
the terrain that the analysis and every downstream visualization read.** A detention
tank is a hole in the ground. If that hole is only a decorative mesh over an
unmodified globe, the water surface, equipment placement, pick coordinates and
analysis grid all disagree — and the tool lies to the planner.

So the excavation is written into the quantized-mesh tile itself. Every Cesium API
that reads terrain then returns the excavated surface automatically.

---

## Submission documents

| | Document | What it covers |
|---|---|---|
| 1 | [Main Project Report (PDF)](docs/1_Main%20Project%20Report.pdf) | Narrative: goal, how it was built, result, future work |
| 2 | [Architecture Document (PDF)](docs/2_Architecture%20Document.pdf) | Diagrams, technical documentation, annotated code excerpts |
| 3 | [Demo Video (MP4)](docs/3_Demo%20Video.mp4) | Walkthrough of the running application |
| 4 | [Screenshots (PNG)](docs/4_Screenshots.png) | Before/after stills of terrain excavation and flood rendering |

---

## Cesium techniques demonstrated

**Custom `TerrainProvider` that patches quantized-mesh tiles.** A delegation
decorator — not a subclass — that forwards the entire `TerrainProvider` contract to
a wrapped provider and intercepts `requestTileGeometry`. When a returned
`QuantizedMeshTerrainData` tile overlaps a registered excavation, its raw
`_quantizedVertices` are handed to a Web Worker for vertex insertion and
re-triangulation, and a fresh `QuantizedMeshTerrainData` is returned into Cesium's
normal geometry pipeline. Bounding volumes are rebuilt with `BoundingSphere` and
`OrientedBoundingBox.fromRectangle` so culling stays correct.

**Selective quadtree invalidation.** `invalidateAllTiles()` works but blanks the
globe for a frame. Instead the quadtree is walked from `_levelZeroTiles` and only
the leaf tiles intersecting the changed excavation are freed, guarded by a fallback
to the public API.

**Time-dynamic flood rendering.** A custom Fabric GLSL material registered through
`Material._materialCache.addMaterial`, applied to a `GroundPrimitive` with
`ClassificationType.TERRAIN`. Two consecutive frame textures are interpolated by a
`lerpT` uniform updated every frame from `viewer.clock` in a `scene.preRender`
listener, so scrubbing the Cesium timeline gives continuous inundation rather than
a slideshow.

**Cesium as the single source of terrain truth.** A close-range facility view is
rendered in a local ENU Three.js scene, but it samples the *excavation-patched*
`viewer.terrainProvider` with `sampleTerrainMostDetailed`, so it inherits the
excavation and cannot drift from the globe.

**Also used:** `ClippingPolygonCollection` on globe and 3D Tiles ·
`WebMapTileService` / `WebMapService` / `UrlTemplate` imagery providers ·
`Cesium3DTileset`, `createOsmBuildingsAsync` · `GroundPolylinePrimitive`,
`RectangleGeometry`, `GeometryInstance`, `PolygonHierarchy` · `GeoJsonDataSource`,
`CzmlDataSource` with clock intervals · `ScreenSpaceEventHandler`,
`scene.pickPosition` · `JulianDate`, `Clock` · self-hosted quantized-mesh terrain
served via `CesiumTerrainProvider.fromUrl`, alongside Cesium ion World Terrain,
World Imagery and OSM Buildings.

87 files in the application import Cesium directly, approximately 30,500 lines.

---

## Note on source access

The codebase is owned by KICT, a Korean government research institute, and is
subject to client confidentiality, so I am not able to grant repository access.
The Architecture Document contains code excerpts reproduced verbatim from the
running codebase, comments included. I am glad to walk through any part of the
implementation on a call.
