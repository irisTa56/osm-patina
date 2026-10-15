# 0005. The features reach the window as vector tiles that tippecanoe builds

- Status: Proposed
- Date: 2026-10-15

## Context

osmium-tool cuts the rectangle out of the extract and writes its features as GeoJSON ([0002](0002-osmium-tool-selects-and-assembles.md)), and walkers draws them as a layer of tiles ([0004](0004-map-drawn-in-rust-in-a-native-window.md)).
In 0004's trial, the program built that layer in memory from the GeoJSON.
The maintainer means the Publishing phase to serve the features as vector tiles built ahead of time, so that a viewer fetches only the tiles in view, and chose for this phase to reach the window by the same kind of tiles (both said 2026-10-15).

- Every rectangle is cut by reading the whole extract, about 5 seconds for the Kanto extract ([phase 01's A007](../plan/phase-01-local-map.md#assumptions--risks)), which tiles built once for a wider area would spare a map whose point moves.
- [tippecanoe](https://github.com/felt/tippecanoe) builds vector tiles from GeoJSON, and since 2.17 writes them as one PMTiles archive (its README, read 2026-10-15).
- None of the tools found that build vector tiles in Rust takes this phase's features as they are (read 2026-10-15):
  - [`tylertoo`](https://github.com/geoparquet-io/tylertoo) reads GeoParquet alone, and its README calls it a work in progress.
  - [`freestiler`](https://github.com/walkerke/freestiler/) has its engine in Rust but is used as an R or Python package.
  - [`elivagar`](https://github.com/folknor/elivagar) builds a base map's layers from PBF in a schema of its own.
- A trial built the stand-in area's 108,074 features, 37 MB of GeoJSON, into archives with tippecanoe 2.79.0, which mise's conda backend installed on macOS arm64, on an Apple M3 (2026-10-15).
  - With its defaults and no simplification at the deepest zoom (`-pS`), 6 features were missing at zoom 14: four small areas and two lines of two points.
  - With small areas kept as they are (`-pt`) and coordinates 16 times finer within a tile (`-d16`), zoom 14 held all 108,074, built in 2.8 seconds into 2.9 MB.
  - With `-pt`, the zoom 13 tile over Tokyo Station could not be made small enough for tippecanoe's limit of 500 KB a tile.
  - At zooms 12 to 15, with no limit on a tile's size, the archive took 3.4 seconds and 8.3 MB.
- The program read that zoom 12 to 15 archive with the [`pmtiles`](https://docs.rs/pmtiles/0.24.0) crate 0.24.0 and [`mvt-reader`](https://docs.rs/mvt-reader/2.5.0) 2.5.0, which walkers 0.61.0 reads PMTiles and vector tiles with, and drew it as 0004's trial did, in two runs.
  - Its first frame came 131 to 355 ms after it started, and 60 tiles were read in 210 to 229 ms.
  - While the map zoomed and moved by itself, the 95th percentile between frames was 17.8 to 17.9 ms.
  - Picking at one point took 0.14 to 0.24 ms, and found another way than 0004's trial did at the same point, since a tile holds each area cut at the tile's edge and the smallest area was judged on the cut ones; which pick is right was not looked at.

## Decision

The command cuts and exports the rectangle with osmium-tool as 0002 says, keeps the features the rules admit, and has tippecanoe build them into a PMTiles archive, which the window reads.
The archive holds every feature whole at zoom 14, with no simplification at that zoom, small areas kept as they are, and coordinates finer than tippecanoe's default.
The map draws the features at zoom 14 and closer, and further out it draws none and says that zooming in shows them.

## Rejected alternatives

- **The features handed from osmium-tool's export to the window, as 0004's trial did**: it needs no tool beyond osmium-tool, but Publishing would replace that path with tiles rather than extend it.
- **Building the tiles in Rust**: it would keep the tools to Rust, but cutting at a tile's edge and encoding vector tiles would be the program's own work, which tippecanoe already does, and no Rust tool found takes the features as they are.
- **Every feature at zooms further out than 14**: a tile over Tokyo Station already goes over tippecanoe's limit at zoom 13, and a 5 km rectangle fits a window of 1,280 pixels at zoom 14, which spans about 10 km at Tokyo's latitude.
- **One archive for a whole region rather than for the rectangle**: it would spare every later rectangle the cut, but how long it takes and how large it is are not measured, which the Location and Publishing phases need to settle anyway.

## Consequences

- **Dependencies added**: tippecanoe, installed by mise's conda backend and run as a separate program, and the `pmtiles` and `mvt-reader` crates.
- **Risks**:
  - conda-forge's build of tippecanoe on Linux has not been installed, which CI would show by failing at install.
  - A feature the archive leaves out or simplifies in a denser area would not be drawn, which the end-to-end test on the fixture and the count of the stand-in area's features would show.
  - Selecting among overlapping areas judges each by the part a tile holds unless the archive carries each feature's whole size, which a test that selects within overlapping areas would show.
  - The rectangle is still cut from the whole extract on every command, until a wider area is built once.
