# 0005. The features reach the window as vector tiles that tippecanoe builds

- Status: Proposed
- Date: 2026-10-15

## Context

osmium-tool cuts the area out of the extract and writes its features as GeoJSON ([0002](0002-osmium-tool-selects-and-assembles.md)), and walkers draws them as a layer of tiles ([0004](0004-map-drawn-in-rust-in-a-native-window.md)).
In 0004's trial, the program built that layer in memory from the GeoJSON.

- Every area is cut by reading the whole extract and exported, about 50 seconds for a prefecture's worth of the Kanto extract ([phase 01's A007](../plan/phase-01-local-map.md#assumptions--risks)).
  - The maintainer judged a wait of several seconds too long each time the map is moved to another place while in use (said 2026-10-14).
  - Tiles built once are opened again without cutting, and the window reads only the tiles in view, reading more as the map moves.
- The maintainer looks at about 5 km in one sitting, zooming in for the smaller features, and wants the map to hold up to about a prefecture, growing by the municipalities added (said 2026-10-24).
- [tippecanoe](https://github.com/felt/tippecanoe) builds vector tiles from GeoJSON, and since 2.17 writes them as one PMTiles archive (its README, read 2026-10-15).
- None of the tools found that build vector tiles in Rust takes this phase's features as they are (read 2026-10-15):
  - [`tylertoo`](https://github.com/geoparquet-io/tylertoo) reads GeoParquet alone, and its README calls it a work in progress.
  - [`freestiler`](https://github.com/walkerke/freestiler/) has its engine in Rust but is used as an R or Python package.
  - [`elivagar`](https://github.com/folknor/elivagar) builds a base map's layers from PBF in a schema of its own.
- Trials with tippecanoe 2.79.0, which mise's conda backend installed on macOS arm64, on an Apple M3:
  - From the 108,074 features of the 5 km square around Tokyo Station, 37 MB of GeoJSON, zooms 14 to 16 with no simplification at the deepest zoom (`-pS`) built in 3.0 seconds into 7.0 MB; zoom 16 held all 108,074, zoom 15 84,040, and zoom 14 74,381 (2026-10-16).
    - Built with zoom 14 as the deepest, 6 of them were missing there, four small areas and two lines of two points; turning tiny-polygon reduction off (`-pt`) brought the areas back, and a detail of 16 (`-d16`) the lines; tippecanoe's README says it reduces tiny polygons at every zoom, the deepest included, unless told not to (read 2026-10-25), and lines finer than a tile's grid collapse (2026-10-15).
  - At zoom 16 alone, squares of 1 cm to 1 m on a side near Tokyo Station: the defaults and `-pS` kept those of 0.5 m and more; adding no tiny-polygon reduction (`-pt`) kept 0.2 m and more; adding a detail of 16 as well (`-d16`, a tile of 65,536 units rather than 4,096) kept every one (2026-10-25).
    - A tagged way of two nodes at one position, and a closed one of four, which `osmium export` writes as lines, came out of no build with `-pS -pt -d16` (2026-10-25); Tokyo's prefecture, cut out of the Kanto extract, held no such way (a scan of 2026-10-25).
  - From a box around Tokyo's mainland cut out of the Kanto extract, every tagged object with no scope rules applied, 9,683,164 features in 3.5 GB of GeoJSON, zooms 10 to 16 with `-pS` failed after 232 seconds, the further zooms' tiles being too large; with `--drop-densest-as-needed` added, which thins a whole zoom where any of its tiles goes over tippecanoe's limit of 500 KB, it built in 612 seconds into 611 MB (2026-10-24).
    - The largest tile was 333 KB at zoom 10, 447 KB at zoom 13, 463 KB at zoom 14, 296 KB at zoom 15, and 137 KB at zoom 16, every tile at the default extent of 4,096.
    - No zoom 16 tile was near the limit, so nothing was thinned there for size; the features at zoom 16 were not counted.
    - Cutting the box took 7.5 seconds and exporting it 45 seconds.
- [`osmium extract`](https://docs.osmcode.org/osmium/latest/osmium-extract.html) cuts by polygons given in an OSM file, merging several (multi)polygons into one area as long as they do not overlap (read 2026-10-24), which distinct municipalities' boundaries do not, though a city and one of its own wards do.
  - Cut by two wards of Tokyo that share an edge, their boundary relations with their ways and nodes in one file, the Kanto extract gave no error and held exactly the objects of the two wards cut one at a time, those reaching across the edge once (osmium 1.19.1, 2026-10-25).
  - Municipalities apart from each other were not tried.
- The program read a zoom 14 archive of that square, built with tippecanoe's default limits, with the [`pmtiles`](https://docs.rs/pmtiles/0.24.0) crate 0.24.0 and [`mvt-reader`](https://docs.rs/mvt-reader/2.5.0) 2.5.0, which walkers 0.61.0 reads PMTiles and vector tiles with, and drew it as 0004's trial did, closer zooms as pieces of zoom 14 tiles, in two runs on an idle machine (2026-10-15).
  - Its first frame came 127 to 377 ms after it started, and the 30 tiles it needed were read in 115 ms.
  - While the map zoomed and moved by itself, the 95th percentile between frames was 17.7 ms.
  - No window was run over an archive of zooms 10 to 16.
  - Picking at one point took 0.36 to 0.44 ms, and found another way than 0004's trial did at the same point; the trial judged the smallest area on each area as the tile holds it, cut at the tile's edge, which may be why, and which pick is right was not looked at.

## Decision

The area is the municipalities given, taken as the union of their boundaries in the extract; osmium-tool cuts and exports it as 0002 says, the features the rules admit are kept, and tippecanoe builds them into one PMTiles archive in one run, which the window reads.
The archive holds zooms 10 to 16; at zoom 10 a window of 1,280 points spans about 160 km at Tokyo's latitude, so a prefecture's worth nearly fits one view.
Zoom 16 is the closest zoom it holds, because there every tile of the box around Tokyo's mainland fitted under tippecanoe's limit with nothing thinned for size, the largest at 137 KB, where zoom 14's had to be thinned; zoom 15 was not tried as the closest.
A closer zoom would hold the same features in four times as many tiles for each zoom, where the window draws closer views by enlarging zoom 16's tiles, as 0004's trial did with zoom 14's; none was tried.
Zoom 16 holds every feature, with no simplification there, but for any tippecanoe drops as too small to draw, which it does at every zoom by default and which the report below shows; which options to build with is left to building.
Further out, tippecanoe thins the features as its README describes (read 2026-10-24), in these ways among others:

- it drops points everywhere, keeping 1 in 2.5 for each zoom out, by default;
- it drops an area too small to draw, or draws it as a small square larger than itself, and drops a line too short to draw, by default;
- and where any tile of a zoom grows too large, it widens the spacing kept between features across that whole zoom, so dense places thin first (`--drop-densest-as-needed`, which it suggests where its defaults alone cannot fit the tiles).

The command that builds the archive compares the features it wrote with those zoom 16 holds and reports any that zoom 16 lacks, since tippecanoe drops features without an error.
Each area carries its whole size, since a tile holds it cut at the tile's edge and the plan's R004 selects the smallest of several areas as a whole.
Adding municipalities builds the archive again over all of them, rather than building the new ones alone and joining their tiles to the old.

## Rejected alternatives

- **Every feature at zoom 14, with tippecanoe's limits lifted**: the whole of a 5 km area would show every feature at once, but it holds every feature at the furthest zoom it draws, where tiles grow with the area's density; the maintainer judged thinning further out natural, since they zoom in for the smaller features (said 2026-10-24).
- **A furthest zoom closer than 10, such as 12**: the archive would build faster and smaller, and its opening view would be lighter, but a prefecture's worth would no longer fit one view; offered both, with zoom 10 recommended, the maintainer chose zoom 10 (said 2026-10-24).
- **Dropping a fraction of every zoom whose tiles grow too large (`--drop-fraction-as-needed`)**, tippecanoe's other suggestion: it would thin sparse and dense places alike, where widening the spacing thins the dense places first; offered both, with the spacing recommended, the maintainer chose it (said 2026-10-24).
- **Building each municipality's archive alone and joining them with `tile-join`**: adding a municipality would build only that one, but a feature crossing a boundary is in both cuts, so each feature would need a rule for which municipality owns it; the thinning further out would differ on either side of a boundary; and a newer extract rebuilds every municipality anyway to bring the dates up; the maintainer agreed to building again (said 2026-10-24).
- **A rectangle of up to 5 km around a chosen point, built and opened one at a time**: each build is quick, but the map holds one rectangle, and growing it brings back the overlap of the previous alternative.
- **osmium-tool's export kept as it is, and read whole by the window, as 0004's trial read it**: it needs no tool beyond osmium-tool, but every opening reads every feature the export holds, which grows with the area, where the window reads from an archive only the tiles in view; offered the tiles and the layer built from the export, the maintainer chose the tiles (said 2026-10-15).
- **No feature missing at zoom 16, held to by tippecanoe's options**: a mapper zoomed in would never miss a feature, but options such as a finer detail make zoom 16's tiles larger and the build longer, by how much not measured over a prefecture, and a way whose nodes lie at one position is dropped under every option tried; what the defaults drop is under a pixel across at zoom 16, where a pixel of a 256-pixel tile spans about 1.9 m at Tokyo's latitude, and the 5 km square's zoom 16 lost none; the maintainer chose it on 2026-10-24 and turned to reporting what is missing on 2026-10-25.
- **Building the tiles in Rust**: it would keep the tools to Rust, but cutting at a tile's edge, thinning, and encoding vector tiles would be the program's own work, which tippecanoe already does, and no Rust tool found takes the features as they are.

## Consequences

- **Dependencies added**: tippecanoe, installed by mise's conda backend and run as a separate program, and the `pmtiles` and `mvt-reader` crates.
- **Risks**:
  - conda-forge's build of tippecanoe on Linux has not been installed, which CI would show by failing at install.
  - Building a prefecture's worth takes about ten minutes, and every municipality added or every newer extract takes that again; reading zoom 16 back to compare it with what was written adds to that.
  - A denser area, or one with finer features, can lose more at zoom 16 than the trials did, which the report shows.
  - An archive is as old as the extract it was built from, which the window shows by the data's date.
