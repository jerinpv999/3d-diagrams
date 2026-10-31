---
name: archflow3d
description: Use when extending, customizing or generating designs for ArchFlow 3D, the single-file React + three.js software-architecture designer (blocks, connector types, groups, RPS and capacity, flow simulation, PNG/GIF export).
---

# ArchFlow 3D

ArchFlow 3D is a single `index.html` app: React 18 (via `htm`, no JSX build) for the UI, and an imperative three.js 0.147 engine (`class ArchScene`) for the 3D stage. There is no bundler. Read `docs/ARCHITECTURE.md` before larger changes.

## Ground rules

- Keep it one file with no build step. New libraries must be UMD builds loaded with a `<script src>` pinned to an exact version on cdnjs or jsDelivr, placed before the inline script.
- Keep three.js at **0.147.0**. It is the last version with `examples/js/controls/OrbitControls.js` as a UMD global. Upgrading means switching to ES modules and a build step.
- React owns data (`diagram = {title,nodes,edges,flows}`); `ArchScene` owns three.js objects. Never mutate three.js objects from React components. Add a method on `ArchScene` and call it.
- Every colour in CSS comes from the tokens at the top of `<style>`. 3D colours come from `CAT`, `GROUP_KINDS`, `BG`, the packet colours and `FLOW_COLORS`.
- Packet colours carry meaning and must stay fixed: `REQ_COLOR` request, `RES_COLOR` response, `EVT_COLOR` one-way event, `STR_COLOR` stream (`DIR_COLOR` maps hop `dir` to them). Never colour packets by flow; a flow's `color` only highlights its path. Keep `FLOW_COLORS` distinct from the packet colours and the CSS `--req/--res/--evt/--str` tokens in sync.
- Docker: the `Dockerfile` vendors the same library versions from npm and rewrites the five CDN `<script src>` URLs with `sed`. If you change a library URL or version in `index.html`, update the `npm install` line and the matching `sed` line, or the build fails its `! grep 'src="https://'` check. New project files must also be added to `PROJECT_FILES`.
- Saved designs must stay backward compatible with `docs/FILE_FORMAT.md` (version 2; version 1 files must still load). If you add fields, make them optional with defaults.

## Where things are

Search `index.html` for these banners:

| Banner | What to edit there |
| --- | --- |
| `catalog` | Block types (`CATALOG`), family colours (`CAT`), group kinds (`GROUP_KINDS`), descriptions (`DESC`), connector kinds (`CONNECTORS`), protocols, packet colours |
| `traffic maths` | `computeLoads`, `capOf`, `utilOf`, `utilColor`, `fmtRps` |
| `templates` | Starter designs (`TEMPLATES`, helpers `N E S G`) and `hopsOf(flow, edges)` |
| `3D helpers` | Block geometry (`buildShape`), labels (`makeLabel`, `makeTag`), export stamp (`drawStamp`) |
| `connector geometry helpers` | `offsetCurve`, `braidCurve`, `dashTexture`, `streamTexture` |
| `scene engine` | `buildEdge` (per-kind geometry), `rebuildGroups`, picking and multi-drag, `spawnHop` (per-kind packets), flow sequencer, live traffic, camera, export rendering |
| `GIF encoder` | Palette (`makeQuantizer`), `lzw`, `encodeGIF` |
| `zip writer` | `makeZip`, `PROJECT_FILES` (update this list when you add docs) |
| `UI` | `Palette`, `Inspector` (block, zone, group, connection, multi-select), `UtilBar`, `TOUR`, `HelpModal`, `ExportMenu`, `App` |

## Recipes

### Add a block type

1. Add `C(type, label, section, family, shape, provider?)` to `CATALOG` and a one-sentence entry to `DESC`.
   - `family` ∈ `client network compute service data messaging ops external`.
   - `shape` must be a `buildShape` case or `'zone'`.
2. If it's a cloud service, set `provider` (`'AWS'`, `'GCP'`, `'AZURE'`) and put it in that provider's `section`.
3. Mention it in the block table in `docs/USER_GUIDE.md`.

### Add a connector kind

1. Add an entry to `CONNECTORS` with `label`, `short` (tag text), `desc` and default `proto`.
2. Draw it in `ArchScene.buildEdge` (a new `kind===` branch). If packets need their own lanes, push curves into `lanes`.
3. Give it packet behaviour in `spawnHop`. If it never replies, skip its response hop in `hopsOf` and give the forward hop its own `dir`.
4. Document it in `docs/USER_GUIDE.md`, `docs/FILE_FORMAT.md` and the Help modal (which lists `CONNECTORS` automatically).

### Add a group kind

Add `{type:'grp-…', label, color}` to `GROUP_KINDS` and a `DESC` entry. The palette, inspector and Help pick it up automatically.

### Change how load or utilization works

`computeLoads` is the single source for req/s on blocks, connections and groups. Thresholds live in `utilColor` (0.7 amber, 0.9 red). Live-traffic density is `ArchScene.visualRate`.

### Add a 3D shape

1. Add a `case 'name':` to `buildShape`. Use `add(g, geometry, material, x, y, z)` with materials `M` (main), `D` (dark), `L` (glow). Set `h` to the shape's top height and `break`.
2. Keep the footprint within about 2.6 × 1.4 units and the height under 2 units.
3. Use the shape in at least one `CATALOG` entry.

### Add a template

1. Add `name: () => ({title, nodes:[...blocks, ...groups], edges:[E(from,to,label,kind)], flows:[{id,name,color,rps,steps:[S(...)]}]})` to `TEMPLATES`. Create groups with `G(id, type, name, memberIds, nodes, desc)` after the blocks; give busy blocks `capacity`/`replicas` and add `desc` where the type's default is too generic.
2. Put zones first in `nodes`. Space blocks at least 3 units apart, laid out left to right: clients → edge → services → data/external.
3. Add an `<option value="name">` to the template `<select>` in `App`.
4. Save the template's JSON into `examples/` and add its path to `PROJECT_FILES`.

### Generate a design for a user's system

Write JSON in the format from `docs/FILE_FORMAT.md`. Use only catalog `type` values. Pick each edge's `kind` from how it really behaves (queues and webhooks are `oneway`, consumers `stream`, gRPC `multiplex`, WebSockets `duplex`). Add one flow per main use case with a realistic `rps`, a `note` for each request and a `ret` for each response. Put co-hosted blocks in a group. The user loads it with **Open JSON**.

### Add a tour step

Add `{sel:'[data-tour=name]', title, body}` to `TOUR` and put `data-tour="name"` on the target element. Steps without `sel` show as a centred card.

### Change export quality

- PNG widths: the `<select id="png-w">` options in `ExportMenu`. Frame shapes are the `frame` options; the size cap is computed in `ArchScene.setTargetSize` (8192 and the GPU's limits).
- GIF widths and frame rates: `gif-w` and `gif-fps` options. Frames render at the GIF's size, so higher widths cost recording time and memory (about 1 byte per pixel per frame).
- Label sharpness: `LK` (label supersample factor, default 2).
- PNG shadow resolution: `setShadowRes(4096)` in `snapshot`.

## Checklist before handing back

1. Extract the inline script and run `node --check` on it.
2. Open `index.html` in a browser. The console has no errors, the checkout template loads, and "Place order" animates.
3. Add a block, drag it, connect it with each connector type, record a 3-step flow and play it. Check packet colours match the legend.
4. Shift-click two blocks, press G, drag the group, ungroup it. Set a capacity and confirm the label colour changes. Turn on Live traffic.
5. Export a 1920 PNG and a 3-second 960 GIF. Both open and are sharp.
6. Save JSON, reload, open the JSON again, and confirm it round-trips.
7. Update the docs (`README.md`, `docs/*.md`) and this file for any user-visible change.
