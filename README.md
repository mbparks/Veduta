# VEDUTA

**A cartographic plate press.** Name a place, and it presses a map poster — on screen, on paper, or on a cutter.

Field Instrument **FI-239** *(number proposed, not yet confirmed against the live catalogue)*
Current version **v2.0.2** · one HTML file, 276 kB · no libraries, no build step, no server

A *veduta* is an eighteenth-century townscape painting — a detailed, faithful view of a city made to hang on a wall. That is what this is for.

---

## What it is

VEDUTA takes a place name or a pair of coordinates, fetches the OpenStreetMap data around it, and turns it into a printable plate. It is a browser instrument: it runs from a single file on your own machine, keeps everything it fetches, and can redraw a plate years later from the bytes it was originally drawn from.

It is the printing sibling of **GRATICULE** (FI-134), which points the same pipeline at a laser bed. Same projection, same Overpass intake, same geometry kernel. Different destination.

## What it is not

- **Not a map viewer.** There is no slippy map, no tile server, no basemap. It draws vectors it has fetched, once, deliberately.
- **Not a GIS.** No reprojection between coordinate systems, no attribute queries, no spatial joins.
- **Not colour-managed.** The ink report separates with plain 100% GCR. It answers "will this flood the sheet past what the press can dry", not "what hue will this be".
- **Not a contouring engine.** Contours and hillshade are *imported*. Deriving them from a DEM is a different instrument.

---

## Quick start

Save `veduta.html` and open it from disk. **Do not run it inside a preview pane or sandboxed iframe** — outbound requests are blocked there and every fetch fails identically. (See *Troubleshooting*.)

The instrument has two doors.

**Easy mode** is what opens first: type a place, pick one of five looks, pick a sheet size, press the plate. It geocodes, fetches, styles, proofs, and hands you a print file. About four seconds of decisions.

**The bench** is the same instrument with the lid off — seven stations, every control exposed. `Bench` in the masthead switches between them, and the easy result has an *Open the bench* button that lands you at Plate with the proof already drawn.

---

## The bench

Numbered because the order is real: you cannot style cargo you have not fetched, or press a plate you have not proofed.

| | Station | What happens there |
|---|---|---|
| 01 | **Fix** | Where the plate is centred. Place lookup, or coordinates — decimal degrees and DMS both parse. |
| 02 | **Extent** | How much ground, how big the sheet, at what resolution. Change either of the first two and the scale re-derives. Margin and bleed live here. |
| 03 | **Cargo** | Fetch the map data once and keep it. Layer picker, request splitting, the manifest, and the file-in path for when the servers are busy. |
| 04 | **Palette** | The ramp — what decides a road's weight and in what ink. Terrain styling and water hatching. |
| 05 | **Plate** | The proof. Drag it to move the fix, scroll to change the ground covered. Cartouche, place names, scale bar, north arrow. |
| 06 | **Press** | One ink at a time. Casings, the ink inventory, separations, and the cut plate. |
| 07 | **Emit** | Pre-flight, then files. Nothing exports until the plate has been proofed from current settings. |

---

## Lineage

VEDUTA began as a port of the two OSMnx scripts in [CarlosLannister/beautifulMaps](https://github.com/CarlosLannister/beautifulMaps). Those are about thirty lines of substance each: fetch a road graph, bin each edge by its length into five colour and width buckets, hand the result to matplotlib on a dark navy ground.

OSMnx and NetworkX are not reimplemented, because the graph abstraction was incidental — the scripts only ever used it as a bag of edges with geometry. VEDUTA fetches ways and relations directly and keeps them as polylines and rings.

Three things are worth knowing about the port.

**The length ramp is faithfully preserved, quirk and all.** Binning by segment length is a rough proxy for importance: a long rural lane outranks a downtown avenue chopped into short blocks. That quirk is exactly what the look is made of, so it ships as a selectable ramp rather than being quietly corrected. The **Madrid Navy** preset carries the original's bin edges and inks unchanged. A truer ramp keyed on the OSM `highway` tag is the default.

**Ways are cut at their junctions first.** OSMnx bins by *graph edge* — intersection to intersection. An OSM way runs through many intersections, so binning raw ways would put a whole arterial in the top bucket and lose the texture entirely. Overpass returns node ids alongside geometry, so VEDUTA counts node degree across the fetched set and cuts ways where two or more meet, reconstructing exactly the edges OSMnx would have built. In testing, 112 grid ways became roughly 6,000 edges. Without this the length ramp is a flat mess.

**The original's finest road is 0.10 pt — 0.035 mm.** That is well under any press hairline; it reads on screen only because the rasteriser antialiases it into a tint. The Madrid preset keeps the number and the ink rule is allowed to complain about it.

Also worth flagging if you go looking at the source: `createWaterMap.py` assigns `point = (...)` and then passes `center_point` to both graph calls. It has never run as committed.

---

## The ink rule

The signature instrument, pinned across the bottom of the chrome at all times.

Stroke widths in VEDUTA are **millimetres on the printed sheet** — not pixels, not points. A width in pixels means nothing until you know the sheet size, and a width in points means nothing to anyone but matplotlib. That choice is what lets the ink rule mean something.

It is a graduated millimetre scale with the press hairline marked, one printer dot marked at the current resolution, and a needle at the narrowest currently-enabled stroke. Below the minimum it goes registration-magenta. It is the one control that connects the abstract number in the ramp table to whether the plate can actually be printed.

---

## Honesty, deliberately

A few places where the instrument tells you something inconvenient rather than guessing.

- **The projection is local equirectangular**, not Web Mercator, so the scale bar is true at the plate. It degrades past about 50 km and the app says so instead of quietly lying at the sheet edges. If you want continent-scale plates, that is a different instrument.
- **A canvas over budget does not throw.** It allocates and then paints nothing, which is how you get a beautiful blank poster. The raster probe paints one corner pixel and reads it back, then tiles if it has to.
- **Water hatching refuses to go below three times the press hairline.** Closer than that the lines merge on the sheet into a solid nobody asked for, so the spacing opens out and the app says it did.
- **A place name that will not fit is dropped and counted**, never clipped.
- **The OpenStreetMap credit is on by default** and pre-flight warns if you switch it off. The ODbL requires attribution on anything derived from the data.
- **The cargo travels in the project file.** Reopen it and the plate redraws with no network at all.

---

## Terrain

Contours and hillshade are imported at Cargo. Both come out of GDAL in one line:

```sh
# hillshade — an ESRI ASCII grid, because it is a header and a wall of numbers
gdal_translate -of AAIGrid dem.tif dem.asc

# contours — GeoJSON, elevation property auto-detected
gdal_contour -a elevation -i 20 dem.tif contours.geojson
```

Neither file is stored in the project. A grid outweighs everything else put together, and it is re-openable from the same file it came from.

The hillshade is Horn's 3×3 slope and aspect with a real degree-to-metre conversion at the grid's latitude, resampled through the same projection as everything else so the ridges land under the roads that follow them. Vertical exaggeration is exposed, because a poster is not a survey and a ridge at true scale reads flat. Gentle relief — Mountain Maryland, say — usually wants more than the default 2.

---

## Exports

| Format | Where | Notes |
|---|---|---|
| **PNG** | Emit | Full plate in one canvas where the browser allows it, otherwise butted tiles plus a montage command. |
| **SVG** | Emit | Real physical `mm` on the root element, paths grouped per ink, cartouche as editable text. |
| **PDF** | Emit | Hand-rolled, no library. MediaBox / TrimBox / BleedBox, DeviceCMYK using the same separation the ink report measures, Flate compression, hillshade embedded as an image XObject with an alpha SMask. |
| **Separations** | Press | One file per ink, drawn on the paper colour rather than the plate background, as SVG or PDF. |
| **Cut plate** | Press | Every stroke of one ink turned to a closed outline and unioned, so a junction is cut once rather than three times. SVG and R12 DXF. |
| **Project** | Emit | JSON carrying the fetched cargo, the style, and the fix. |

The DXF is **R12 (AC1009)** with polylines written the R12-legal way as `POLYLINE` / `VERTEX` / `SEQEND`. `LWPOLYLINE` is R14 and up and a strict reader refuses it inside an AC1009 file; maximum compatibility is the whole point of choosing R12.

---

## The geometry kernel

A Martinez and Rueda sweep-line boolean — union, intersection, difference, xor, with holes nested by containment — plus stroke-to-outline and polygon offsetting. Ported from the one written for GRATICULE.

Stroked polylines become geometry by emitting one convex quad per segment plus a convex join at each vertex, then unioning the lot. Building a continuous offset path uses fewer primitives but every join is a special case, and a miter join has to *replace* the previous offset end and the current start — emit both and you leave a zero-area spike that costs nothing in area and wrecks the boolean. Convex pieces have no such trap.

Five things the tests found, all invisible to an area check:

1. **`compareSegments` is not symmetric.** It resolves ties assuming the first argument is the segment being inserted. Call it the other way round and a vertical segment sorts above the horizontal it is about to cross, admitting interior edges into every union.
2. **The contour walk must decrement *past* the start index**, not down to it. Stopping at it leaves the walk on an already-processed event, circling one contour until the array overflows.
3. **Collinear vertices in the output poison the next boolean.** A run of points along one line gives the sweep several zero-turn segments to order against each other and the union starts fragmenting. Results are stripped so they can be fed straight back in.
4. **The final normalisation cannot be `boolop(rings, [], UNION)`.** That takes the trivial shortcut and hands every ring back as its own outer, silently losing every hole.
5. **Erosion is not dilation with the sign flipped.** Offset a ring inward past its own half width and it turns itself inside out. Shrink is done as the complement of the dilation of the complement, so small shapes vanish instead of leaving scrap.

### Cutting a street grid

The cut path is **chain → simplify → dither → stroke → union**, and the order is the whole trick. Drop any one of the three preparatory steps and a city grid comes back as dozens of malformed contours.

- **Chain.** Roads are cut at their junctions to serve the length ramp. For cutting that split is actively harmful: two collinear pieces butt against each other along a shared edge exactly one stroke wide, and an exact collinear overlap between two *different* polygons is the least forgiving input a sweep line takes. Pieces are chained back into whole streets, continuing straightest through each crossing rather than turning the corner.
- **Simplify.** A chained street inherits an interior point at every junction. A collinear point carries no shape but earns a join disc, and a disc at every crossing is what the sweep chokes on.
- **Dither.** Everything remaining is nudged by a nanometre, keyed on the *coordinate* so that roads meeting at a junction move together and the junction survives. Derived from the coordinate rather than random, so the same plate always cuts the same way.

A 24 × 24 split grid — 1,104 pieces into 48 streets — resolves to exactly one contour with 529 holes in 89 ms. A real plate cuts in about 100 ms.

The negative control is kept as a live assertion rather than a comment: if the unchained case ever starts working, the test fails and tells you the chaining may no longer be needed.

---

## Testing

Everything runs in the page. Two suites, both under Emit and Press:

- **Self-test** — 123 assertions covering coordinate parsing, projection round-trips, clipping, relation assembly and hole nesting, coastline closing, label placement, separation arithmetic, PDF encoding, DEM parsing, hillshade, hatching, and the export contracts.
- **Kernel self-test** — 30 assertions on the boolean, stroke-to-outline, offsetting, the grid case, and the DXF flavour.

Outside the page, for development:

```sh
npm install jsdom
node harness.js     # 103 assertions — boots the file, drives the UI, hostile input
node ktest.js       # 26 — boolean areas against hand-worked values
node stest.js       # 21 — stroke-to-outline and offsetting
node gridtest.js    # the street-grid case, both with and without chaining
node e2e.js         # a synthetic town end to end, three presets, out to SVG
node terrain.js     # contours, DEM, hillshade, hatching, out to PDF
node cuttest.js     # casings, separations, cut plate
```

Independent validation where it matters: `pypdf` and `poppler` on the PDF (boxes, even-odd fills, embedded image and SMask byte counts, rendered output), `cairosvg` on the SVG, `ezdxf` conventions on the DXF.

Three encoding bugs surfaced only under an independent reader and are worth repeating for anyone writing a PDF by hand: **the content stream is not UTF-8** (a simple font is one byte per glyph, so a degree sign written as two UTF-8 bytes images as two wrong glyphs); **document-info strings are not WinAnsi** and want UTF-16BE with a BOM; and **`Td` positions relatively**, so a second pass over the same string for a knockout halo images it alongside the first rather than on top — use `Tm`.

---

## Troubleshooting

**`Failed to fetch` at Cargo.** That is a `TypeError`, not an HTTP status — the request never left the browser and Overpass never saw it. No mirror will help. In order of likelihood: the page is running inside a sandboxed preview (save it and open from disk); a content blocker is catching the request; or there is no route to those hosts. The place lookup at Fix is the quick test — it is a different host, so if that fails too it is the sandbox or the network, and if it works it is a blocker. **Check the connection** at Cargo probes every instance and tells you which.

**A fetch that sits there.** Cancel is available in both doors and there is a 75-second abort. The status line names the instance, the method, and the elapsed seconds.

**HTTP 406.** The main `overpass-api.de` instance has added scraper defences that turn ordinary tool traffic away. It is deliberately last in the endpoint list, behind `overpass.kumi.systems`, `overpass.private.coffee`, and the `z.` and `lz4.` instances.

**Nothing at all comes back.** Use **Show the query** at Cargo, run it at [overpass-turbo.eu](https://overpass-turbo.eu), export raw JSON, and drop it into the file-in path. The plate does not care where the bytes came from.

**Be a decent citizen.** The public instances are volunteer-run and under real strain. If VEDUTA becomes something you use regularly rather than occasionally, run a local instance covering the states you care about — a few gigabytes — and point the file-in path at its output.

---

## Version history

| | |
|---|---|
| **v1.0** | Six stations, the ink rule, both ramps, junction splitting, SVG and tiled raster with the canvas probe. |
| **v1.1** | Cartographic correctness — multipolygon holes nested by containment, coastline closed against the plate box, place names with knockout halos and collision avoidance. |
| **v1.2** | Press readiness — hand-rolled PDF, measured ink coverage, crop marks and registration targets, media/trim/bleed geometry. |
| **v1.3** | Terrain — contour import, hillshade from an ESRI grid, water hatching held above minimum separation. |
| **v1.4** | The front door — easy mode, drag and scroll on the proof, cargo coverage checking, scale bar and north arrow. |
| **v2.0** | The geometry kernel — casings, per-ink separations, the cut plate, R12 DXF. |
| **v2.0.1** | Endpoint mirrors, GET fallback, request timeouts, and failure classification that distinguishes a block from a refusal. |
| **v2.0.2** | Front-door progress reporting and cancellation; request splitting thresholds corrected; file-only layers excluded from the query. |

### Still open

- The FI-239 number is proposed and unconfirmed.
- No PWA manifest.
- Nothing is persisted between sessions — which door you last used, or your press settings.
- A Gears of Resistance post, likely covering GRATICULE and VEDUTA together, since the laser-and-press pairing is the more interesting story than either alone.

---

## Licence and attribution

Map data © OpenStreetMap contributors, available under the [Open Database Licence](https://www.openstreetmap.org/copyright). Anything you produce with VEDUTA is a derived work and must carry that credit — the cartouche prints it by default, and pre-flight will tell you if you have switched it off.

Geocoding by [Nominatim](https://nominatim.openstreetmap.org/). Map data by the [Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API) and its mirrors.

Concept and lineage: [beautifulMaps](https://github.com/CarlosLannister/beautifulMaps) by Carlos Lannister.

---

*Make. Hack. Learn. Share. Repeat.*
