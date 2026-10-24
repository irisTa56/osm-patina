# 0005. The features reach the window as vector tiles that tippecanoe builds

- Status: Proposed
- Date: 2026-10-15

## Context

osmium-tool cuts the area out of the extract and writes its features as GeoJSON ([0002](0002-osmium-tool-selects-and-assembles.md)), and walkers draws them as a layer of tiles ([0004](0004-map-drawn-in-rust-in-a-native-window.md)).
In 0004's trial, the program built that layer in memory from the GeoJSON.

- Every area is cut by reading the whole extract and exported, about 50 seconds for a prefecture's worth of the Kanto extract ([phase 01's A007](../plan/phase-01-local-map.md#assumptions--risks)), and the roadmap's scope has the map opened at the user's location or a point picked on it later ([roadmap](../ROADMAP.md#scope)).
  - The maintainer judged a wait of several seconds each time the point is set or moved while the map is in use too long (said 2026-10-14).
  - Tiles built once are opened again without cutting, and anywhere within them.
- The maintainer looks at about 5 km in one sitting, zooming in for the smaller features, and wants the map to hold up to about a prefecture, growing by the municipalities added (said 2026-10-24).
- [tippecanoe](https://github.com/felt/tippecanoe) builds vector tiles from GeoJSON, and since 2.17 writes them as one PMTiles archive (its README, read 2026-10-15).
- None of the tools found that build vector tiles in Rust takes this phase's features as they are (read 2026-10-15):
  - [`tylertoo`](https://github.com/geoparquet-io/tylertoo) reads GeoParquet alone, and its README calls it a work in progress.
  - [`freestiler`](https://github.com/walkerke/freestiler/) has its engine in Rust but is used as an R or Python package.
  - [`elivagar`](https://github.com/folknor/elivagar) builds a base map's layers from PBF in a schema of its own.
- Trials with tippecanoe 2.79.0, which mise's conda backend installed on macOS arm64, on an Apple M3:
  - From the 108,074 features of the 5 km square around Tokyo Station, 37 MB of GeoJSON, zooms 14 to 16 with no simplification at the deepest zoom (`-pS`) built in 3.0 seconds into 7.0 MB; zoom 16 held all 108,074, zoom 15 84,040, and zoom 14 74,381 (2026-10-16).
  - From a box around Tokyo's mainland cut out of the Kanto extract, every tagged object with no scope rules applied, 9,683,164 features in 3.5 GB of GeoJSON, zooms 10 to 16 with `-pS` failed after 232 seconds, the further zooms' tiles being too large; with `--drop-densest-as-needed` added, which thins only a tile over tippecanoe's limit of 500 KB, it built in 612 seconds into 611 MB (2026-10-24).
    - Every zoom 16 tile was under 137 KB and at the default extent of 4,096, so nothing was thinned there; the features at zoom 16 were not counted.
    - Cutting the box took 7.5 seconds and exporting it 45 seconds.
- [`osmium extract`](https://docs.osmcode.org/osmium/latest/osmium-extract.html) cuts by polygons given in an OSM file, merging several (multi)polygons into one area as long as they do not overlap (read 2026-10-24), which distinct municipalities' boundaries do not, though a city and one of its own wards do; not run, and how it takes municipalities that share an edge was not tried.
- The program read a zoom 14 archive of that square, built with tippecanoe's default limits, with the [`pmtiles`](https://docs.rs/pmtiles/0.24.0) crate 0.24.0 and [`mvt-reader`](https://docs.rs/mvt-reader/2.5.0) 2.5.0, which walkers 0.61.0 reads PMTiles and vector tiles with, and drew it as 0004's trial did, closer zooms as pieces of zoom 14 tiles, in two runs on an idle machine (2026-10-15).
  - Its first frame came 127 to 377 ms after it started, and the 30 tiles it needed were read in 115 ms.
  - While the map zoomed and moved by itself, the 95th percentile between frames was 17.7 ms.
  - No window was run over an archive of zooms 10 to 16.
  - Picking at one point took 0.36 to 0.44 ms, and found another way than 0004's trial did at the same point; the trial judged the smallest area on each area as the tile holds it, cut at the tile's edge, which may be why, and which pick is right was not looked at.

## Decision

The area is the municipalities given, taken as the union of their boundaries in the extract; osmium-tool cuts and exports it as 0002 says, the features the rules admit are kept, and tippecanoe builds them into one PMTiles archive in one run, which the window reads.
The archive holds zooms 10 to 16, every feature at zoom 16, with no simplification at that zoom, and further out tippecanoe thins the features: it drops points everywhere, keeping 1 in 2.5 for each zoom out, as it does by default, and where a zoom's tiles grow too large it widens the spacing kept between features across that zoom, so dense places thin first (`--drop-densest-as-needed`, which tippecanoe suggests where its defaults alone cannot fit the tiles; its README, read 2026-10-24).
The command that builds the archive compares the features it wrote with those zoom 16 holds and reports any that zoom 16 lacks, since tippecanoe drops features without an error.
Each area carries its whole size, since a tile holds it cut at the tile's edge and the plan's R004 selects the smallest of several areas as a whole.
Adding municipalities builds the archive again over all of them, rather than building the new ones alone and joining their tiles to the old.

## Rejected alternatives

- **Every feature at zoom 14, with tippecanoe's limits lifted**: the whole of a 5 km area would show every feature at once, but it holds every feature at the furthest zoom it draws, where tiles grow with the area's density; the maintainer judged thinning further out natural, since they zoom in for the smaller features (said 2026-10-24).
- **Dropping a fraction of every zoom whose tiles grow too large (`--drop-fraction-as-needed`)**, tippecanoe's other suggestion: it would thin sparse and dense places alike, where widening the spacing thins the dense places first; offered both, with the spacing recommended, the maintainer chose it (said 2026-10-24).
- **Building each municipality's archive alone and joining them with `tile-join`**: adding a municipality would build only that one, but a feature crossing a boundary is in both cuts, so each feature would need a rule for which municipality owns it; the thinning further out would differ on either side of a boundary; and a newer extract rebuilds every municipality anyway to bring the dates up; the maintainer agreed to building again (said 2026-10-24).
- **A rectangle of up to 5 km around a chosen point, built and opened one at a time**: each build is quick, but the map holds one rectangle, and growing it brings back the overlap of the previous alternative.
- **osmium-tool's export kept as it is, and read whole by the window, as 0004's trial read it**: it needs no tool beyond osmium-tool, but every opening reads every feature the export holds, which grows with the area, where the window reads from an archive only the tiles in view; offered the tiles and the layer built from the export, the maintainer chose the tiles (said 2026-10-15).
- **Building the tiles in Rust**: it would keep the tools to Rust, but cutting at a tile's edge, thinning, and encoding vector tiles would be the program's own work, which tippecanoe already does, and no Rust tool found takes the features as they are.

## Consequences

- **Dependencies added**: tippecanoe, installed by mise's conda backend and run as a separate program, and the `pmtiles` and `mvt-reader` crates.
- **Risks**:
  - conda-forge's build of tippecanoe on Linux has not been installed, which CI would show by failing at install.
  - Building a prefecture's worth takes about ten minutes, and every municipality added or every newer extract takes that again; reading zoom 16 back to compare it with what was written adds to that.
  - An archive is as old as the extract it was built from, which the window shows by the data's date.
