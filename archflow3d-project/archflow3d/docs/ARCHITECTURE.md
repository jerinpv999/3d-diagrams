# ArchFlow 3D code architecture

The app is one file, `index.html`. It has no build step and no bundler. Four pinned UMD libraries are loaded from CDNs, then one inline script runs the app.

| Library | Version | Global |
| --- | --- | --- |
| React | 18.3.1 | `React` |
| ReactDOM | 18.3.1 | `ReactDOM` |
| htm | 3.1.1 | `htm` (JSX-like tagged templates, bound to `React.createElement`) |
| three.js | 0.147.0 | `THREE` |
| OrbitControls | 0.147.0 (examples/js) | `THREE.OrbitControls` |

three.js 0.147 is the last line that still ships `examples/js` UMD controls, which is why it is pinned there.

## Layers

```
┌──────────────────────── React (UI state) ────────────────────────┐
│  App ── diagram {title, nodes[], edges[], flows[]}                │
│   │     sel, mode, rec, playing, hop, speed, loop, ambient …      │
│   ├─ Palette        ├─ Inspector       ├─ ExportMenu              │
│   ├─ Tour           ├─ HelpModal                                   │
│   └─ <div canvas-host>  ← ArchScene mounts its WebGL canvas here   │
└────────────┬───────────────────────────────▲──────────────────────┘
   setData(nodes,edges)                       │ callbacks
   setSelection / setMarked                   │ onPick(hit)
   playFlow / stopFlow / speed / ambient      │ onMove(id,x,z)
   view / snapshot / startCapture             │ onHop(i) / onFlowEnd()
┌────────────▼───────────────────────────────┴──────────────────────┐
│  ArchScene (imperative three.js engine)                            │
│   renderer · camera · OrbitControls · lights · ground grid         │
│   nodes Map<id,{group,h,sig}> · edges Map<id,{group,curve,hit}>    │
│   packets[] · pulses[] · flowRun · capture                         │
│   tick(): sequencer → ambient → packets → pulses → render → capture│
└────────────────────────────────────────────────────────────────────┘
```

React owns the **data**. `ArchScene` owns the **3D objects** and animation. React never touches three.js objects directly; it calls the scene's methods, and the scene reports user actions back through callbacks. This keeps 60 fps animation out of React's render cycle.

### Why not React Three Fiber?

R3F has no maintained UMD build, so using it would need a bundler. A thin imperative engine keeps the project a single file you can open from disk. The separation above maps cleanly onto R3F if you want to port it: each `buildShape` case becomes a component, `ArchScene.tick` becomes `useFrame`, and `setData` disappears because R3F reconciles the scene from props. See *Porting to React Three Fiber* below.

## Code map (search for these banners in `index.html`)

| Banner | Contents |
| --- | --- |
| `catalog` | `CAT`, `CATALOG`, `GROUP_KINDS`, `DESC` (built-in descriptions), `CONNECTORS`, `PROTOCOLS`, packet colours (`REQ_COLOR`, `RES_COLOR`, `EVT_COLOR`, `STR_COLOR`), `FLOW_COLORS`, `BG` |
| `traffic maths` | `edgeBetween`, `computeLoads`, `capOf`, `utilOf`, `utilColor`, `fmtRps` |
| `templates` | `TEMPLATES` with `N()` node, `E(from,to,label,kind)` edge, `S()` flow step, `G()` group helpers; `hopsOf(flow, edges)` |
| `3D helpers` | `buildShape(shape,color)`, `makeLabel`, `makeTag`, `glowTex`, `drawStamp`, `disposeTree` |
| `connector geometry helpers` | `offsetCurve`, `braidCurve`, `dashTexture`, `streamTexture` |
| `scene engine` | `class ArchScene` |
| `GIF encoder` | `makeQuantizer`, `lzw`, `encodeGIF` |
| `zip writer` | `crc32`, `makeZip`, `PROJECT_FILES`, `selfHTML` |
| `saving` | `getDownloads`, `saveFile`, `saveError`, `slug` |
| `persistence` | `loadLocal`, `saveLocal` (localStorage key `archflow3d.v1`) |
| `UI` | `Palette`, `NumField`, `UtilBar`, `MultiPanel`, `Inspector`, `TOUR`, `Tour`, `HelpModal`, `ExportMenu`, `cleanGroups`, `App` (tooltip via `showTip`) |

## Data model

See [FILE_FORMAT.md](FILE_FORMAT.md). In short: nodes have a `type` that keys into `CATALOG`; edges join node ids; flows are ordered lists of node ids with notes.

## The scene engine

### Sync

`setData(diagram, {labelMode})` receives the whole design. It computes loads with `computeLoads`, then diffs blocks by id. Each block stores a signature (type, name, size, description, capacity, replicas, rounded load, label mode); if it changes, the block's mesh and label are rebuilt (labels are textures). Positions are applied directly. Edges and groups are cheap and rebuilt on every sync.

### Traffic maths

`computeLoads(d)` returns `{node, edge}` maps of req/s. A block gets the `rps` of every flow that visits it (once per flow); an edge gets the `rps` of every flow that crosses it in either direction; a group gets the sum of its members. `capOf(n) = capacity × replicas`, `utilOf` divides, `utilColor` maps to green / amber (≥ 0.7) / red (≥ 0.9). Labels show load and utilization, and blocks at 90% or more get a red ground ring.

### Picking and dragging

A capture-phase `pointerdown` on the host element runs before OrbitControls. If the pointer is over a block it disables the controls, so dragging moves the block instead of orbiting. Hits are prioritised: blocks, then connections (an invisible fat tube), then group floors, then zones. A press that moves less than 5 px is a click and calls `onPick(hit, {shift})`; shift-click builds a multi-selection (`{kind:'multi', ids}`). A drag moves one block, the whole multi-selection, or every member of a dragged group, snapping the shared offset to 0.5, and calls `onMoveMany` once on release.

### Groups

Group nodes (`type` starting `grp-`) have no geometry of their own. `rebuildGroups()` finds members (`node.group === id`), takes their live positions and heights, and builds a padded floor plate, faint walls (not pickable), an outline box and a label hanging below the front edge. It runs on every sync, selection change and drag frame, so boxes follow their members. React's `cleanGroups()` removes groups that lose all members.

### Edges and connector kinds

`curveBetween(a,b,side)` makes a quadratic Bézier from block A to block B. `buildEdge` then draws it according to `kind`:

| Kind | Geometry | `lanes` |
| --- | --- | --- |
| `request` | Tube, arrow at end | none |
| `oneway` | Tube with a repeating dash `alphaMap` (`alphaTest` 0.5) | none |
| `twoway` / `halfduplex` | Tube, arrows at both ends; half duplex adds a disc at the midpoint | none |
| `duplex` | Two tubes from `offsetCurve(±0.17)`, opposite arrows | 2 (forward, backward) |
| `multiplex` | Three helices from `braidCurve` around a faint core | 3 |
| `stream` | Tube with a gradient `map` whose offset scrolls every frame | none |

Side curves taper to zero offset at both ends so lanes meet at the blocks. Tags are cached textures keyed by text.

### Flow sequencer and live traffic

`hopsOf(flow, edges)` returns forward hops (`dir` = `req`, or `evt` over one-way links, or `stream` over streaming links) followed by response hops, skipping one-way and streaming links. `playFlow` steps through them one at a time, highlights the flow's edges in its colour (`setHighlight`), and reports `onHop(i)` for the Hop log.

`spawnHop(hop)` turns a hop into packets according to the connector: one packet; a packet per multiplex lane with staggered starts; a train of four for streams; or, on duplex links, a packet on the matching lane plus a simultaneous response packet on the other. Colours come from `DIR_COLOR` and never from the flow.

With `live = true`, each flow accumulates `visualRate(rps) = min(5, 0.35 + 0.9·log10(rps+1))` journeys per second (capped at 80 in flight). Each journey runs its own hop list with small packets.

### Packets and pulses

A packet is a sphere with an additive glow sprite and a short trail. Its travel time scales with curve length, with ease-in-out. A pulse briefly scales the block and expands a fading ring on the ground.

## Export pipeline

### PNG

`snapshot(width, aspect)` calls `setTargetSize(width, aspect)`, which sets the pixel ratio to 1 and resizes the drawing buffer to exactly `width × width/aspect` (aspect defaults to the stage's), updating the camera aspect to match. The canvas keeps its on-screen CSS size. Width is capped by `MAX_RENDERBUFFER_SIZE`, `maxTextureSize` and 8192. If auto-fit is on, `exportFit()` saves the camera and calls `fitPrecise(dir, 0.9)`: it binary-searches the camera distance along the current view direction until the projected bounds of every block footprint and label fit inside ±0.9 of the frame, re-centring the target each pass. The **Fit** button uses the same routine. Shadows switch to 4096 px for the render. The frame is copied to a 2D canvas, the title stamp is drawn, and `toBlob('image/png')` produces the file. The renderer is then restored.

Labels are drawn on canvases at 2× (`LK = 2`) with mipmaps, so they hold up at 4K and 8K.

### GIF

`startCapture({seconds,fps,width,aspect,…})` sets the buffer to the GIF size, so each frame is a 1:1 copy with no resampling blur. On each capture tick:

1. The frame is copied and stamped.
2. On the first frame, `makeQuantizer` builds a 256-colour palette by median cut over a 5-bit histogram, with exact colour sums so palette entries are true averages. It also adds weighted samples for every flow colour, the response colour and their blends toward the background and white. That keeps packets and glows clean even though they may not appear in the first frame.
3. The frame is mapped to palette indices through a lazily filled 6-bit LUT (262,144 entries) and stored as a `Uint8Array`. Only indices are kept, about 1 byte per pixel instead of 4.

`encodeGIF` writes a GIF89a with a global palette, a NETSCAPE loop block and one LZW-compressed image per frame. It yields between frames so the progress bar updates.

### Saving files

`saveFile(name, blob)` uses the hosting viewer's `downloads` capability when present (`window.claude.use('downloads')`), which shows a confirmation dialog. When the app runs as a plain file or on your own server, it falls back to an `<a download>` link.

### Project zip

`makeZip` writes an uncompressed (stored) zip with CRC-32 and UTF-8 names. `downloadZip` fetches each path in `PROJECT_FILES` from `project/` (the hosted layout) or `./` (your own server). If `index.html` can't be fetched, it serialises the running page with `selfHTML()`. The current design is always added as `designs/<title>.json`.

## Extending

### Add a block type

Add one line to `CATALOG`:

```js
C('cassandra','Cassandra','Data','data','db'),
//  type        label      section  family shape  [provider]
```

- `section` controls which palette group it appears in. A new section name creates a new group.
- `family` (`cat`) picks the colour from `CAT`.
- `shape` picks the 3D model from `buildShape`. Use `'zone'` for a resizable area.
- Add a provider (`'AWS'`, `'GCP'`, `'AZURE'`, or your own) to show it on the label.

### Add a 3D shape

Add a `case` to `buildShape`. Build meshes with `add(group, geometry, material, x, y, z)`, using `M` (main), `D` (dark) and `L` (light/glow) materials, and set `h` to the top height so the label sits above it. Keep the footprint within about 2.6 × 1.4 units so blocks don't overlap at the default spacing.

### Add a template

Add a function to `TEMPLATES` returning `{title, nodes, edges, flows}` using `N`, `E` and `S`, then add an `<option>` to the template `<select>` in `App`.

### Add a protocol suggestion

Add it to `PROTOCOLS`. Labels are free text, so this only affects the suggestion list.

### Change the look

CSS tokens are at the top of the `<style>` block (`--ink`, `--panel`, `--req`, `--res`, fonts). The 3D background is `BG`. Family colours are in `CAT`.

## Porting to React Three Fiber

1. Create a Vite React app and install `three`, `@react-three/fiber` and `@react-three/drei`.
2. Move `CATALOG`, `TEMPLATES`, `hopsOf`, the GIF encoder and the zip writer into plain modules unchanged.
3. Turn `buildShape` cases into `<Shape kind color />` components returning meshes.
4. `<Node>` renders a shape plus drei `<Billboard>` with a texture label (or drei `<Text>`). Dragging can use `@use-gesture/react` with a ground-plane raycast.
5. `<Edge>` builds the same Bézier with `useMemo` and renders `<mesh><tubeGeometry args={[curve,48,0.05,8]} /></mesh>`.
6. Packets live in a ref array advanced inside `useFrame`.
7. For exports, resize the renderer with `preserveDrawingBuffer: true` exactly as `setTargetSize` does.
